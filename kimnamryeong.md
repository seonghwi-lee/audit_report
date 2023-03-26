# DEX - removeLiquidity 최소 토큰 발급량 등호 실수

## 설명

<aside>

**키워드 : Condition mistake**

**심각도 : Informational**

등호 실수로 인해 지정한 최솟값과 동일한 양의 토큰 출금하려고 시도하면 revert가 발생한다.

</aside>

```solidity
require(
            amountX > _minimumTokenXAmount && amountY > _minimumTokenYAmount,
            "INSUFFICIENT_LIQUIDITY_BURNED"
        );
```

최소 지정 값과 동일한 양의 토큰이 지급될 상황에 `>=` 가 아닌 `>` 로 검사하여 revert가 발생한다.

## 파급력

지급되어야 할 토큰이 지급되지 않는 상황이 발생하고 revert되어 인출 실패 및 가스 소모만 발생하는 상황이 연출될 수 있다. 하지만 유저의 손해로만 이어질 뿐, 다시 호출할 수 있고 최솟값을 파라미터로 지정하여 넘겨줄 수 있기에 악용의 여지는 없기에 위험도는 없다고 판단하였다.

## 해결 방법

```solidity
require(
            amountX >= _minimumTokenXAmount && amountY >= _minimumTokenYAmount,
            "INSUFFICIENT_LIQUIDITY_BURNED"
        );
```

등호를 추가하면 된다.
