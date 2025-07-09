# tarea-modulo3
Este proyecto consta de tres contratos desarrollados en Solidity:

- TokenA y TokenB: Representan tokens ERC-20 personalizables que se pueden mintear por el propietario.
- SimpleDEX: Un exchange descentralizado simplificado que permite intercambiar TokenA por TokenB (y viceversa), así como agregar y remover liquidez.
📦 TokenA.sol
TokenA es un contrato ERC-20 que representa un token llamado “TokenA” con símbolo “TKA”. Solo el propietario (owner) puede crear nuevos tokens.

Herencias:
- ERC20: Proporciona las funciones estándar del token ERC-20.
- Ownable: Restringe ciertas funciones al propietario del contrato.

Código:
```
pragma solidity ^0.8.22;

import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";

contract TokenA is ERC20, Ownable {
    constructor() ERC20("TokenA", "TKA") Ownable(msg.sender) {
        _mint(msg.sender, 1000 * 10 ** decimals());
    }

    function mint(address to, uint256 amount) public onlyOwner {
        _mint(to, amount);
    }
}
```

📦 TokenB.sol
TokenB es un contrato ERC-20 similar a TokenA, pero representa un token distinto llamado “TokenB” con símbolo “TKB”.

Código:
```
pragma solidity ^0.8.22;

import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";

contract TokenB is ERC20, Ownable {
    constructor() ERC20("TokenB", "TKB") Ownable(msg.sender) {
        _mint(msg.sender, 1000 * 10 ** decimals());
    }

    function mint(address to, uint256 amount) public onlyOwner {
        _mint(to, amount);
    }
}
```

🔁 SimpleDEX.sol
SimpleDEX es un contrato para intercambiar dos tokens ERC-20 entre sí y administrar un pool de liquidez.

Dependencias:
- IERC20: Interfaz para tokens ERC-20.
- Ownable: Restringe funciones administrativas.

Código Principal:
```
pragma solidity ^0.8.22;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract SimpleDEX is Ownable {
    IERC20 public tokenA;
    IERC20 public tokenB;
    
    uint256 public reserveA;
    uint256 public reserveB;
    
    uint256 private constant FEE_NUMERATOR = 997;
    uint256 private constant FEE_DENOMINATOR = 1000;
    
    event LiquidityAdded(address indexed provider, uint256 amountA, uint256 amountB);
    event LiquidityRemoved(address indexed provider, uint256 amountA, uint256 amountB);
    event Swapped(address indexed user, string direction, uint256 inputAmount, uint256 outputAmount);
    
    constructor(address _tokenA, address _tokenB) Ownable(msg.sender) {
        require(_tokenA != address(0) && _tokenB != address(0), "Direcciones invalidas");
        tokenA = IERC20(_tokenA);
        tokenB = IERC20(_tokenB);
    }
    
    function addLiquidity(uint256 _amountA, uint256 _amountB) external onlyOwner {
        require(_amountA > 0 && _amountB > 0, "Cantidad invalida");
        require(tokenA.transferFrom(msg.sender, address(this), _amountA), "Fallo en transferencia de TokenA");
        require(tokenB.transferFrom(msg.sender, address(this), _amountB), "Fallo en transferencia de TokenB");
        reserveA += _amountA;
        reserveB += _amountB;
        emit LiquidityAdded(msg.sender, _amountA, _amountB);
    }
    
    function removeLiquidity(uint256 _amountA, uint256 _amountB) external onlyOwner {
        require(_amountA <= reserveA && _amountB <= reserveB, "Reservas insuficientes");
        reserveA -= _amountA;
        reserveB -= _amountB;
        require(tokenA.transfer(msg.sender, _amountA), "Fallo al transferir TokenA");
        require(tokenB.transfer(msg.sender, _amountB), "Fallo al transferir TokenB");
        emit LiquidityRemoved(msg.sender, _amountA, _amountB);
    }
    
    function swapAforB(uint256 _amountAIn) external {
        require(_amountAIn > 0, "Cantidad invalida");
        require(tokenA.transferFrom(msg.sender, address(this), _amountAIn), "Fallo en transferencia");
        uint256 amountBOut = _getSwapOutput(_amountAIn, reserveA, reserveB);
        reserveA += _amountAIn;
        reserveB -= amountBOut;
        require(tokenB.transfer(msg.sender, amountBOut), "Fallo al transferir TokenB");
        emit Swapped(msg.sender, "A->B", _amountAIn, amountBOut);
    }
    
    function swapBforA(uint256 _amountBIn) external {
        require(_amountBIn > 0, "Cantidad invalida");
        require(tokenB.transferFrom(msg.sender, address(this), _amountBIn), "Fallo en transferencia");
        uint256 amountAOut = _getSwapOutput(_amountBIn, reserveB, reserveA);
        reserveB += _amountBIn;
        reserveA -= amountAOut;
        require(tokenA.transfer(msg.sender, amountAOut), "Fallo al transferir TokenA");
        emit Swapped(msg.sender, "B->A", _amountBIn, amountAOut);
    }
    
    function getPrice() external view returns (uint256 priceAperB, uint256 priceBperA) {
        require(reserveA > 0 && reserveB > 0, "Sin liquidez");
        priceAperB = (reserveB * 1e18) / reserveA;
        priceBperA = (reserveA * 1e18) / reserveB;
    }
    
    function _getSwapOutput(uint256 inputAmount, uint256 inputReserve, uint256 outputReserve) internal pure returns (uint256) {
        require(inputReserve > 0 && outputReserve > 0, "Reservas vacias");
        uint256 inputWithFee = inputAmount * FEE_NUMERATOR;
        uint256 numerator = inputWithFee * outputReserve;
        uint256 denominator = (inputReserve * FEE_DENOMINATOR) + inputWithFee;
        return numerator / denominator;
    }
}
```
✨ Funciones principales
 - addLiquidity: Agrega liquidez al pool, solo ejecutable por el owner.

 - removeLiquidity: Retira liquidez del pool.

 - swapAforB / swapBforA: Permite intercambiar un token por otro.

 - getPrice: Devuelve el precio relativo entre los dos tokens.

 - _getSwapOutput: Calcula el resultado de un swap aplicando una comisión del 0.3%.

📊 Eventos
LiquidityAdded: Emite detalles cada vez que se agrega liquidez.

LiquidityRemoved: Emite detalles al remover liquidez.

Swapped: Emite información de cada intercambio de tokens.

🔒 Seguridad y Restricciones
Solo el propietario puede mintear tokens y gestionar la liquidez.

Las transferencias y los swaps están protegidos por validaciones de monto y reservas.

Comisiones aplicadas a cada intercambio (0.3%) para prevenir ataques de arbitraje o manipulación.
