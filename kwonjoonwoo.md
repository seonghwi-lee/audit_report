## DEX - addLiquidity의 유동성 공급비 검증의 부재

### 설명

<aside>
💡 **키워드 : Vulnerability - 유동성 풀 조작
심각도 : Critical**
`addLiquidity` 함수내부에서 두 토큰의 유동성 공급량 비율에 대한 검증이 이루어지지 않는다.
이를 악용하여 유동성 풀에 원하는 비율로 토큰을 공급하여 교환비를 조작할 수 있다.

</aside>

```solidity
if (liquiditySum == 0) {
    lpToken = Math.sqrt(tokenXAmount * tokenYAmount); //initial token amount
} else {
    (X, Y) = update();

    // 기존 토큰에 대한 새 토큰의 비율로 계산
    uint liquidityX = (liquiditySum * tokenXAmount) / X;
    uint liquidityY = (liquiditySum * tokenYAmount) / Y;
    lpToken = (liquidityX < liquidityY) ? liquidityX : liquidityY;
}
_tokenX.transferFrom(msg.sender, address(this), tokenXAmount);
_tokenY.transferFrom(msg.sender, address(this), tokenYAmount);
transfer(msg.sender, lpToken);
```

토큰 발급은 더 적은 토큰 공급량에 맞추기 때문에 이를 악용하여 DEX의 유동성을 탈취하는 행위는 불가능하다.

하지만 기존 비율과 맞지 않는 유동성을 공급할 수 있고, 이는 전체 유동성 풀의 비율을 깨뜨릴 수 있게 된다.

### PoC

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
Running 1 test for test/Dex.t.sol:DexTest
[PASS] testPOC() (gas: 422523)
Logs:
  LP1 10000000000000000000
  LP2 100000000000000000
  user 2 : before swap balance 0 101000000000000000000
  user 2 : after swap balance 50474737368684342171 0
  user 2 : Attack result 50974987493746873436 2000000000000000000
```

swap의 결과를 보면 알 수 있듯 100:1 의 가치로 교환이 되어야 하지만 유동성 풀의 비율이 101:101 로 깨진 상태이기에 Y토큰 101 ether로 X 토큰 약 50 ether를 교환한 것을 확인할 수 있다.

하지만 토큰 발행량은 더 적은 쪽인 Y토큰에 맞춰 발행되었기에  토큰 수는 실제 공급량보다 적게 발행되었기에 공격을 시도하더라도 공격자가 이득을 취할 수 있는 방법은 존재하지 않는 것으로 보인다.

### 파급력

개인이든 집단이든 공격을 통해 유동성풀의 비율을 깨뜨려 교환비를 변경할 수는 있지만, 공격을 통해 깨진 유동성 풀의 비율로 인한 이득은 기존에 유동성풀을 공급한 모두가 공유하는 구조이기에 공격을 위해 투자한 비용을 모두 회수하며 추가적인 이득을 보는 방법은 존재하지 않는다.

피싱이나 요청을 조작하는 등 제 3자가 잘못된 비율로 유동성공급을 하도록 유도한 뒤 변화한 유동성을 이용하여 토큰을 변환하거나 유동성을 제거하여 이득을 취할 수는 있지만 이를 직접 유동성의 변화를 만든뒤 이득을 취하는 용도로 악용하는 방도는 없어 보인다.

그렇다고 할지라도 DEX의 중심 기능이자 핵심인 유동성 풀의 비율을 자본을 투입하여 망가뜨릴 수 있다는 것은 DEX의 가용성을 해친다고도 볼 수 있기에 위험도는 매우 위험하다고 판단하였다.

### 해결 방법

```solidity
require(
    tokenXAmount * amountY == tokenYAmount * amountX,
    "imbalance add liquidity"
);
```

를 추가하여 비율이 맞지 않으면 revert 시키거나 비율에 맞는 만큼만 공급량에 추가하도록 로직을 수정해야 한다.

## DEX - addLiquidity시 유동성 풀과의 오차 발생

### 설명

<aside>
💡 **키워드 : Vulnerability - 유동성 수치 오류
심각도 : High**
`addLiquidity` 시 현재 유동성에서 수치를 가져오지 않고 로컬 변수에서 가져오게 되는데, `swap` 함수 실행시 로컬 변수를 제대로 업데이트 하지 않아 이전 유동성 풀 비율을 사용하게 된다.

</aside>

```solidity
// addLiquidity 내부
uint256 xBalance = balances[address(tokenX)];
uint256 yBalance = balances[address(tokenY)];

// swap 내부
tokenY.transferFrom(msg.sender, address(this), tokenYAmount);
tokenX.transfer(msg.sender, outputAmount);
```

`addLiquidity` 시 토큰으로부터 잔액을 참조하지 않고 로컬 변수로부터 값을 가져온다.

`removeLiquidity` 실행 시 로컬 변수를 제대로 업데이트 하지만, `swap` 실행 과정에서는 로컬 변수를 업데이트해주는 로직이 빠져있어 `swap` 이후 실행되는 `addLiquidity` 는 실제 유동성 풀과의 비율에 차이가 발생한다.

이로 인해 현재 가격이 아닌 이전 공급량을 기준으로 토큰을 발급받게 되고, 이는 유동성 공급자의 손해로 이어지게 된다.

### PoC

```solidity
function testPOC() external {
    dex.addLiquidity(100 ether, 100 ether, 0);
    tokenY.transfer(address(0x01), 90 ether);
    vm.startPrank(address(0x01));
    {
        tokenY.approve(address(dex), 90 ether);
        dex.swap(0 ether, 90 ether, 0);
        console.log(
            "swap result",
            tokenX.balanceOf(address(0x01)),
            tokenY.balanceOf(address(0x01))
        );
    }
    vm.stopPrank();
    dex.addLiquidity(1 ether, 1 ether, 0);
}
```

```solidity
Running 1 test for test/Dex.t.sol:DexTest
[PASS] testPOC() (gas: 249860)
Logs:
  balanec 0 0
  real balance 0 0
  swap result 47343478489810963087 0
  balanec 100000000000000000000 100000000000000000000
  real balance 52656521510189036913 190000000000000000000
```

실제 balance는 `swap` 에 의해 비율이 깨졌지만, add에서는 balances 변수에 접근하여 값을 가져오기에 서로 싱크가 맞지 않아 잘못된 양의 토큰을 발급받게 된다. 하지만 후술하겠지만 `transferFrom` 을 통한 토큰 제공 부분이 구현되어 있지 않아 악용 가능성은 없다.

### 파급력

토큰들의 실제 가치가 변하여도 swap을 통해 유동성 풀에는 적용이 되지만, 로컬 변수에는 removeLiquidity를 통해서만 변화가 적용되기에 시간이 지날수록 실제 가치와는 괴리가 생기게 될 것이다.

이로 인해 유동성 공급자는 공급한 가치보다 더 적은 양의 토큰을 발급받기에 손해를 보게 될 것이고, 이로 인한 이득은 swap을 통해 실제 가치에 맞게 변환하는 유저들이 이득을 취하게 될 것이다.

그러나 이 공격으로 얻을 수 있는 이득은 공격을 위해 지불한 공급량과 같거나 그보다 적은 양의 토큰으로 이득을 볼 수 있기에 해당 공격으로 이득을 취할 수는 없다. 단, 제 3자는 이를 통해 망가진 교환비를 사용하여 이득을 얻을 수 있다.

전체 로직에 사용되는 유동성 풀이 망가진 것은 아니지만, 이로 인해 유동성 공급자들이 계속해서 피해를 보게 되며 이 수치는 시간이 갈 수록 커질 것이기에(swap이 호출될때마다 괴리가 발생) 위험도가 높다고 판단하였다.

## DEX - removeLiquidity 토큰 전송로직 미구현

### 설명

<aside>
💡 **키워드 : 현금화 불가
심각도 : Critical**
얼마의 토큰을 전송할지에 대한 반환값은 제시하지만 실제 x토큰과 y토큰을 전송하지 않기에 유동성을 제거할 수 없다.(LP토큰을 현금화할 수 없다.)

</aside>

```solidity
function removeLiquidity(
        uint256 LPTokenAmount,
        uint256 minimumTokenXAmount,
        uint256 minimumTokenYAmount
    ) external returns (uint, uint) {
        require(
            LPTokenAmount <= LPToken_balances[msg.sender],
            "Insufficient LP token balance"
        );
        require(LPTokenAmount > 0, "Invalid LP token amount");
        uint xBalance = (LPTokenAmount * tokenX.balanceOf(address(this))) /
            totalLiquidity;
        uint yBalance = (LPTokenAmount * tokenY.balanceOf(address(this))) /
            totalLiquidity;
        require(
            xBalance >= minimumTokenXAmount && yBalance >= minimumTokenYAmount,
            "Insufficient liquidity"
        );
        balances[address(tokenX)] -= xBalance;
        balances[address(tokenY)] -= yBalance;
        console.log("maybe", xBalance / 1e18, yBalance / 1e18);
        LPToken_balances[msg.sender] -= LPTokenAmount;
        totalLiquidity -= LPTokenAmount;
        return (xBalance, yBalance);
    }
```

`transferFrom` 을 통해 실제 x,y 토큰을 전송하는 로직이 없어 유동성 공급시 발급하는 LP토큰이 실제 가치로 교환되지 않고 삭제되기만 한다.

### PoC

```solidity
function testPOC() external {
        dex.addLiquidity(100 ether, 100 ether, 0);

        tokenX.transfer(address(0x02), 1000 ether);
        tokenY.transfer(address(0x02), 100 ether);
        uint lp;
        vm.startPrank(address(0x02));
        {
            tokenX.approve(address(dex), 1000 ether);
            tokenY.approve(address(dex), 100 ether);
            lp = dex.addLiquidity(1000 ether, 100 ether, 0);

            console.log(
                "before remove lquidity ",
                lp,
                tokenX.balanceOf(address(0x02)),
                tokenY.balanceOf(address(0x02))
            );
            dex.removeLiquidity(lp, 0, 0);
            console.log(
                "after remove lquidity ",
                lp,
                tokenX.balanceOf(address(0x02)),
                tokenY.balanceOf(address(0x02))
            );
        }
        vm.stopPrank();
    }
```

```solidity
Running 1 test for test/Dex.t.sol:DexTest
[PASS] testPOC() (gas: 261851)
Logs:
  before remove lquidity  100000000000000000000 0 0
  after remove lquidity  100000000000000000000 0 0
```

유동성을 제거하여도 토큰만 줄어들고 실제 토큰으로 교환되지 않음을 확인할 수 있다.

### 파급력

유동성 공급시에는 정상적으로 토큰을 LP토큰으로 변환해준다. 하지만 이후 유동성을 제거하며 실제 토큰으로 변환할 때에는 LP토큰만 제거하고 실제 토큰을 지급하지 않는다.

이는 LP토큰의 가치가 없다고 판단할 수 있고, DEX의 유동성 공급이 만들어질 수 없는 요인이 되며, 손해는 고스란히 유동성 공급자들이 받게 된다.

이러한 현상은 유동성 공급을 억제하고 DEX의 핵심 기능을 해치게 된다고 판단하여 위험도를 매우 높음으로 매겼다.

### 해결 방법

```solidity
tokenX.transfer(msg.sender, xBalance);
tokenY.transfer(msg.sender, yBalance);
```

`transfer` 를 사용하여 x토큰과 y토큰을 LP토큰 지분에 맞게 지급하도록 작성한다.
