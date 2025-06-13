## 📄 Contrato `MinteoCondicionalSimple`

Este contrato inteligente permite el **minteo condicional** de NFTs tipo ERC-1155 con metadatos personalizados **on-chain**. Solo permite mintear si la wallet que ejecuta la función (`msg.sender`) **posee al menos un NFT** en un contrato externo específico, el cual se pasa por parámetro.

---

### 🎯 Objetivo

Garantizar que solo puedan mintear ciertos NFTs aquellas cuentas que hayan recibido previamente al menos un NFT desde otro contrato. Esta lógica puede utilizarse para sistemas de promoción, acceso restringido, o campañas de reconocimiento.

---

### ⚙️ Funcionalidades

- ✅ **Validación de tenencia previa**  
  Recorre los `tokenId` del contrato externo (0 a 99) y verifica que la wallet que llama tenga al menos uno.

- 🔐 **Condición de acceso al minteo**  
  Si la wallet no califica, se revierte la transacción con un error personalizado `WalletNoCalifica`, lo cual puede ser interceptado fácilmente desde el frontend.

- 🧾 **Minteo con metadata on-chain**  
  Al mintear, se almacena on-chain un conjunto de campos descriptivos:
  - `titulo`
  - `descripcion`
  - `nombre`
  - `fecha`
  - `imageUrl`

- 🔁 **Evento `TransferSingle`**  
  Emite un evento compatible con la especificación ERC-1155 para facilitar la indexación y consulta de transferencias.

- 🔍 **Función `getMetadata(uint256)`**  
  Permite consultar los metadatos asociados a cada NFT minteado.

- 🧪 **Función `uri(uint256)`**  
  Devuelve una URI con metadatos en formato JSON embebido (`data:application/json`) que puede ser interpretada por navegadores o plataformas NFT.

---

### 🧠 Requisitos del contrato externo

El contrato externo debe implementar al menos esta interfaz:

```solidity
interface ICustomERC1155 {
    function balanceOf(address account, uint256 id) external view returns (uint256);
}


// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;


/// @title MinteoCondicionalSimple
/// @notice Permite mintear un NFT si la wallet que llama tiene al menos 1 NFT en el contrato pasado por parámetro

interface ICustomERC1155 {
    function balanceOf(address account, uint256 id) external view returns (uint256);
}

contract MinteoCondicionalSimple {
    uint256 public currentTokenId = 0;

    struct Metadata {
        string titulo;
        string descripcion;
        string nombre;
        string fecha;
        string imageUrl;
    }

    mapping(uint256 => Metadata) public metadatas;
    mapping(address => mapping(uint256 => uint256)) public balances;

    event TransferSingle(address indexed operator, address indexed from, address indexed to, uint256 id, uint256 value);

    /// @dev Error personalizado que puede ser capturado en el frontend
    error WalletNoCalifica();

    constructor() {}

    function mintConMetadataCondicional(
        address to,
        address contratoVerificacion,
        string memory titulo,
        string memory descripcion,
        string memory nombre,
        string memory fecha,
        string memory imageUrl
    ) public {
        ICustomERC1155 contrato = ICustomERC1155(contratoVerificacion);

        bool tieneAlMenosUno = false;

        for (uint256 id = 0; id < 100; id++) {
            try contrato.balanceOf(msg.sender, id) returns (uint256 balance) {
                if (balance > 0) {
                    tieneAlMenosUno = true;
                    break;
                }
            } catch {
                continue;
            }
        }

        if (!tieneAlMenosUno) {
            revert WalletNoCalifica();
        }

        uint256 tokenId = currentTokenId;
        balances[to][tokenId] = 1;

        metadatas[tokenId] = Metadata({
            titulo: titulo,
            descripcion: descripcion,
            nombre: nombre,
            fecha: fecha,
            imageUrl: imageUrl
        });

        emit TransferSingle(msg.sender, address(0), to, tokenId, 1);
        currentTokenId++;
    }

    function getMetadata(uint256 tokenId) public view returns (
        string memory titulo,
        string memory descripcion,
        string memory nombre,
        string memory fecha,
        string memory imageUrl
    ) {
        Metadata memory data = metadatas[tokenId];
        return (
            data.titulo,
            data.descripcion,
            data.nombre,
            data.fecha,
            data.imageUrl
        );
    }

    function uri(uint256 tokenId) public view returns (string memory) {
        return string(abi.encodePacked(
            "data:application/json;utf8,{\"name\":\"NFT #",
            toString(tokenId),
            "\",\"description\":\"On-chain metadata NFT\",",
            "\"image\":\"", metadatas[tokenId].imageUrl, "\"}"
        ));
    }

    function toString(uint256 value) internal pure returns (string memory) {
        if (value == 0) return "0";
        uint256 temp = value;
        uint256 digits;
        while (temp != 0) {
            digits++;
            temp /= 10;
        }
        bytes memory buffer = new bytes(digits);
        while (value != 0) {
            digits -= 1;
            buffer[digits] = bytes1(uint8(48 + uint256(value % 10)));
            value /= 10;
        }
        return string(buffer);
    }
}
