# 1. DEX - addLiquidity 초기 공급 시 round down

## 설명

<aside>

### **키워드 : Numeric issue(Round down)**

### **심각도 : Informational**

초기 유동성 공급 과정에서 내림(round down)이 발생할 수 있고, 이로 인해 LP토큰 발행량과 실제 유동성 풀의 싱크가 깨지게 되어 이후 유저들 또한 손해를 볼 수 있다.

</aside>

```solidity
_decimal = 10 ** 18;
if (totalSupply() == 0) {
            lpAmount = (tokenXAmount * tokenYAmount) / _decimal; // is amount best? no overflow?
        }
```

초회 공급시 공급된 토큰들의 전체 가치를 `MINIMUM_LIQUIDITY` 상수로 나누어 저장한다. 이 과정에서 999 wei 이하는 모두 삭제되어 LP Token이 발행된다.

따라서 초회 유동성 공급시 공급량에 비해 적은 양의 토큰을 발급받게 된다.

이후 유동성을 공급하는 유저들도 LP토큰 발행량과 유동성 풀에 존재하는 수치의 싱크가 맞지 않기에(유동성 풀이 round down되어 사라진 만큼 항상 큰 상태가 유지되기 때문에) 유동성을 제거할 때 토큰의 가치가 적게 책정된다.

## PoC

```solidity
function testPOC() external {
        uint lp1 = dex.addLiquidity(1 ether + 1, 1 ether + 1, 0);
        emit log_named_uint("LP1", lp1);
        tokenX.transfer(address(0x01), 100 ether);
        tokenY.transfer(address(0x01), 100 ether);
        vm.startPrank(address(0x01));
        {
            tokenX.approve(address(dex), 100 ether);
            tokenY.approve(address(dex), 100 ether);
            uint lp2 = dex.addLiquidity(10 ether, 10 ether, 0);
            emit log_named_uint("LP2", lp2);
            (uint rx, uint ry) = dex.removeLiquidity(lp2, 0, 0);
            emit log_named_uint("lp2:rx", rx);
            emit log_named_uint("lp2:ry", ry);
        }
        vm.stopPrank();
        (uint rx, uint ry) = dex.removeLiquidity(lp1, 0, 0);
        emit log_named_uint("rx", rx);
        emit log_named_uint("ry", ry);
    }
```

```solidity
[PASS] testPOC() (gas: 298446)
Logs:
  LP1: 1000000000000000002
  LP2: 10000000000000000009
  lp2:rx: 9999999999999999999
  lp2:ry: 9999999999999999999
  rx: 1000000000000000002
  ry: 1000000000000000002
```

발행된 토큰 수와 유동성 풀이 일치하지 않기에 공급량을 제거할 때 토큰이 실제 가치보다 낮게 책정됨을 알 수 있다.

## 파급력

초기 공급량(두 토큰 공급량의 곱, `(tokenXAmount * tokenYAmount)` )이 `_decimal` 보다 작다면 발생할 수 있는 버그이다. 발현 난이도는 낮은 편이라고 볼 수 있다.

이후 LP 토큰을 교환하는 모든 유저들에게 손해가 고스란히 가게 되어 있는 구조이지만 이를 악용하여 손해만 볼 뿐 추가적인 이득을 취할 수는 없으며, 손해로 직결되는 양 또한 적은 양이므로 위험도는 낮음으로 책정하였다.

## 해결 방법

```solidity
lpAmount = (tokenXAmount * tokenYAmount)
```

`_decimal` 로 나누지 않거나 overflow가 걱정된다면 `sqrt` 로 연산하여 저장하는 방법이 존재한다.

---

# 2. DEX - addLiquidity 유동성 공급비 검증 미흡

## 설명

<aside>

### **키워드 : Vulnerability - Numeric issue(Round down), 유동성 풀 조작**

### **심각도 : Critical**

`addLiquidity` 함수내부에서 두 토큰의 유동성 공급량 비율에 대한 검증이 이루어지지만, 검증이 미흡하여 유동성 풀을 조작할 수 있다.

</aside>

```solidity
require(
    (_decimal * tokenXAmount) / tokenYAmount ==
        (_decimal * _amountX) / _amountY,
    "amount breaks the pool ratio"
);
```

비율에 대한 검사가 이루어지지만, 결국 나누기 연산에 의해 round down이 발생할 수 있고, 검증의 정확도가 떨어지게 된다. 특히 두 토큰의 전체 공급량이 `10^18` 배 보다 많이 차이가 나는 순간 비율이 0이 되어 유동성의 비와 맞지 않아도 `10^18` 배 이상의 어떠한 공급도 허용하게 된다.

그 이후 토큰 공급량을 계산하는 과정에서 `tokenXAmount` 로부터 지분을 계산하기 때문에 이를 악용하여 토큰 탈취가 가능하다.

`lpAmount = (totalSupply() * tokenXAmount) / _amountX;`

## PoC

```solidity
function testPOC() external {
    dex.addLiquidity(1, 10 ether, 0);
    uint lp1 = dex.addLiquidity(99, 100 ether, 0);

    (uint r1, uint r2) = dex.removeLiquidity(lp1, 0, 0);
    console.log("res", r1, r2 / 1e18);
}
```

```solidity
Running 1 test for test/Dex.t.sol:DexTest
[PASS] testPOC() (gas: 219863)
Logs:
  res 99 108
```

유동성 풀이 `10^18` 배 차이나는 상황에서는 이를 우회하여 유동성을 공급할 수 있고, 이렇게 유동성을 공급하게 될 경우 토큰 발행량은 X에 맞춰지기 때문에 이를 통해 실제 유동성에 공급한 공급량보다 더 많은 토큰 지분을 가지게 된다. 발행받은 토큰을 다시 유동성을 제거하여 토큰 x,y로 바꿔보면 DEX에 존재하는 대부분의 지분(99/100)만큼을 획득하였음을 알 수 있다.

## 파급력

실제 공급한 공급량보다 더 많은 LP토큰을 획득할 수 있는 취약점이기에, 발현할 수 있는 조건만 갖춰진다면 DEX 유동성 풀의 비율도 망가뜨리고 원하는 만큼 토큰도 탈취할 수 있는 취약점이다.

물론 취약점 발현을 위한 조건은 꽤 어려운 편이라고 볼 수 있다. 두 토큰의 교환비(가치)가 `10^18` 배 이상 차이가 발생해야 하기 때문이다.

하지만, 취약점이 발현될 경우 DEX의 핵심 기능이라고 볼 수 있는 가치 교환비가 깨지게 되고 DEX 유동성 풀을 탈취하는 등 DEX를 완전히 무너뜨릴 수 있기 때문에 위험도가 매우 높다고 판단하였다.

## 해결 방법

```solidity
require(
    _amountX * tokenYAmount == tokenXAmount * _amountY,
    "amount breaks the pool ratio"
);
```

round down이 발생하지 않도록 나눗셈이 아닌 곱셈을 통해 비율을 확인하면 해결 가능하다.
