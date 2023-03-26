# DEX - swap 오타

## 설명

<aside>

### **키워드: typo, meaningless logic, gas waste**

### **심각도 : Informational**

오타로 인해 유동성 풀을 표기하는 변수가 잘못 설정되지만, 이후 이 값을 사용하는 로직이 존재하지 않아 악용의 여지는 없음

</aside>

`swap` 함수에서 유동성 풀을 업데이트 하는 과정에서 오타 존재.

```solidity
// swap함수의 100번째 줄
// tokenXAmount가 0일 때 실행되는 코드
tokenY_in_LP += tokenXAmount;
```

원래 로직대로면 `tokenYAmount` 를 더해야 맞는 로직인데 tokenXAmoun를 더했다.

해당 분기에서는 `tokenXAmount` 가 0이기에 유동성 풀의 수치를 나타내는 tokenY_in_LP이 변화하지 않아 원래 실행되어야 하는 로직과는 다른 로직이 실행된다.

하지만 이후 로직에서 `tokenY_in_LP` 를 사용하는 로직이 존재하지 않으며, 다른 함수를 실행할 때에도 진입 부분에서 모두

```solidity
tokenX_in_LP = tokenX.balanceOf(address(this));
tokenY_in_LP = tokenY.balanceOf(address(this));
```

를 호출하여 값을 업데이트 하기에 현재 함수들에서 악용의 여지는 없다.

다만, 전역변수로 설정한 뒤 사용은 지역변수처럼 하기에 전역변수로 선언하여 가스 소모가 늘어났으며, 의미없는 로직이기에 가스 소모만 발생한다.

## 해결방법

유동성 풀을 나타내는 변수(`tokenX_in_LP` , `tokenY_in_LP` )를 전역변수가 아닌 지역변수로 선언하여 사용한다.

유동성풀에 변화가 발생한 경우에는 다시 balanceOf로 데이터를 받아와 사용하고 별도의 처리 로직을 없앤다.
