---
title: BA逆向资源与解包思路
date: 2025/05/01
categories: 
- 游戏开发
tags:
- python
---

#### 前言

* BA最新的游戏资源是在日服, 所以通常想获取需要: 安装安卓日服BA, 注册日服账号, 进入游戏, 才会动态拉取游戏所有资源, 总之很笨重繁琐
* 我需要一个轻量的方法来获取最新的游戏美术资源: 直接从服务器按需获取资源



#### 流程

* 获取最新资源服务器地址, 这一步是难点, 放在本文最后一节

  * 当前2025年5月份日服最新版本的资源服务器地址为: https://prod-clientpatch.bluearchiveyostar.com/r79_swwin27e1u6czsvrxshi_3

  

* 直接浏览器访问`服务器地址`+`Android/bundleDownloadInfo.json`即可获取到所有BundleAssets的资源信息列表, 此时我们可以从里面按需获取资源ID, 例如我想要小春(koharu)的资源, 就直接搜`koharu`, 如果只要3D资源搜`koharu_original`, 资源信息如下(其中Name字段便是资源ID): 

  * 完整地址: https://prod-clientpatch.bluearchiveyostar.com/r79_swwin27e1u6czsvrxshi_3/Android/bundleDownloadInfo.json

```json
[
    {
      "Name": "assets-_mx-characters-koharu_original-_mxdependency_anim_assets_all_3786658315.bundle",
      "Size": 5208436,
      "IsPrologue": false,
      "Crc": 3786658315,
      "IsSplitDownload": false
    },
    {
      "Name": "assets-_mx-characters-koharu_original-_mxdependency_assets_all_3546548379.bundle",
      "Size": 536439,
      "IsPrologue": false,
      "Crc": 3546548379,
      "IsSplitDownload": false
    },
    {
      "Name": "assets-_mx-characters-koharu_original-_mxdependency_mat_assets_all_3814452055.bundle",
      "Size": 4809,
      "IsPrologue": false,
      "Crc": 3814452055,
      "IsSplitDownload": false
    },
    {
      "Name": "assets-_mx-characters-koharu_original-_mxdependency_prefab_assets_all_3453201862.bundle",
      "Size": 69441,
      "IsPrologue": false,
      "Crc": 3453201862,
      "IsSplitDownload": false
    },
    {
      "Name": "assets-_mx-characters-koharu_original-_mxdependency_psd_assets_all_478833476.bundle",
      "Size": 346879,
      "IsPrologue": false,
      "Crc": 478833476,
      "IsSplitDownload": false
    },
    {
      "Name": "character-koharu_original-_mxload-2024-05-30_assets_all_629783588.bundle",
      "Size": 90217,
      "IsPrologue": false,
      "Crc": 629783588,
      "IsSplitDownload": false
    },
]
```

* 直接访问`服务地址`+ `/Android/`+ `资源ID`即可下载对应的资源文件
  * 例如: https://prod-clientpatch.bluearchiveyostar.com/r79_swwin27e1u6czsvrxshi_3/Android/assets-_mx-characters-koharu_original-_mxdependency_assets_all_3546548379.bundle



* 到这一步其实游戏的资源文件就已经拿到了, 剩下AssetBundle解包流程就简单了, 直接选好所有下载好的资源文件, 拖到AssetStudio(AssetStudio的版本必须用最新不然会解析错误)
  * AssetStudio下载地址: https://github.com/Perfare/AssetStudio/releases



* 然后从Asset List找需要的资源即可, 例如想要3D模型则直接导出`Koharu_Original_Mesh`, 如图所示:

![ba1_1746635891131.png](https://s2.loli.net/2025/05/09/NgzyGenj2VU5pZS.png)

* 最后把导出的fbx文件导入到blender测试下, 记得FBX Unit的单位要改为m, (默认值为cm)

![ba1_1746635989334.png](https://s2.loli.net/2025/05/09/f6oNlLG29vjT4dK.png)

* 效果如下, 完工

![ba1_1746636083853.png](https://s2.loli.net/2025/05/09/eGrczSxVjgD8yk3.png)





#### 获取最新资源服务器地址流程

1. 访问: https://prod-noticeindex.bluearchiveyostar.com/prod/index.json, 搜索`LatestClientVersion`, 看到对应的值为`1.56`, 这是最新的客户端版本
2. 访问地址获取1.56版本的游戏客户端: https://d.apkpure.com/b/XAPK/com.YostarJP.BlueArchive?nc=arm64-v8a&sv=24&versionCode=56
3. 解压apk, 到资源路径`./assets/bin/Data`下执行脚本, 脚本会打印服务器地址, 原理是通过UnityTool扫该目录下的所有AssetBundle来解析

```python
# 打印资源服务器地址

import time
import base64
import json
from base64 import b64decode
import os

import UnityPy
from UnityPy.files.File import ObjectReader
from Crypto.Util.strxor import strxor
from xxhash import xxh32_intdigest


def calculate_hash(name: bytes | str) -> int:
    """Calculate a 32-bit hash using xxhash with UTF-8 encoding if needed."""
    if isinstance(name, str):
        name = name.encode("utf8")
    return xxh32_intdigest(name)


def create_key(name: str, size: int = 8) -> bytes:
    """Create a random key based on a hashed name and a specific size."""
    seed = calculate_hash(name)
    return MersenneTwister(seed).next_bytes(size)


def xor(value: bytes, key: bytes) -> bytes:
    """XOR operation between two byte arrays."""
    if len(value) == len(key):
        return strxor(value, key)
    if len(value) < len(key):
        return strxor(value, key[: len(value)])
    # Handle the case where the value is longer than the key
    return b"".join(
        strxor(value[i: i + len(key)], key)
        for i in range(0, len(value) - len(key) + 1, len(key))
    ) + strxor(
        value[len(value) - len(value) % len(key):], key[: len(value) % len(key)]
    )


def convert_string(value: bytes | str, key: bytes = b"") -> str:
    """Decrypt or decode a base64 string or raw bytes, depending on the input."""
    if not value:
        return ""

    try:
        raw = b64decode(value)
        if decoded := xor(raw, key).decode("utf16"):
            return decoded
        raise UnicodeError
    except:
        if isinstance(value, bytes):
            return value.decode("utf8")

    return ""


class MersenneTwister:
    # Constants for the Mersenne Twister algorithm
    N = 624
    M = 397
    MATRIX_A = 0x9908B0DF  # Constant vector a
    UPPER_MASK = 0x80000000  # Most significant w-r bits
    LOWER_MASK = 0x7FFFFFFF  # Least significant r bits

    def __init__(self, seed: int | None = None) -> None:
        if seed is None:
            seed = int(time.time() * 1000)  # Use current time in milliseconds as seed
        self.mt = [0] * self.N  # Create an array to store the state
        self.mti = self.N + 1  # Initial value for mti
        self.init_genrand(seed)

    def init_genrand(self, seed: int) -> None:
        """Initializes the generator with a seed."""
        self.mt[0] = seed & 0xFFFFFFFF  # Seed is limited to 32 bits
        for i in range(1, self.N):
            self.mt[i] = (
                                 1812433253 * (self.mt[i - 1] ^ (self.mt[i - 1] >> 30)) + i
                         ) & 0xFFFFFFFF
        self.mti = self.N

    def _generate_numbers(self) -> None:
        """Generates N words at a time."""
        for i in range(self.N - self.M):
            y = (self.mt[i] & self.UPPER_MASK) | (self.mt[i + 1] & self.LOWER_MASK)
            self.mt[i] = (
                    self.mt[i + self.M] ^ (y >> 1) ^ (self.MATRIX_A if y % 2 else 0)
            )
        for i in range(self.N - self.M, self.N - 1):
            y = (self.mt[i] & self.UPPER_MASK) | (self.mt[i + 1] & self.LOWER_MASK)
            self.mt[i] = (
                    self.mt[i + (self.M - self.N)]
                    ^ (y >> 1)
                    ^ (self.MATRIX_A if y % 2 else 0)
            )
        y = (self.mt[self.N - 1] & self.UPPER_MASK) | (self.mt[0] & self.LOWER_MASK)
        self.mt[self.N - 1] = (
                self.mt[self.M - 1] ^ (y >> 1) ^ (self.MATRIX_A if y % 2 else 0)
        )
        self.mti = 0

    def genrand_int32(self) -> int:
        """Generates a random number on [0, 0xFFFFFFFF]-interval."""
        if self.mti >= self.N:
            self._generate_numbers()

        y = self.mt[self.mti]
        self.mti += 1

        # Tempering transformation
        y ^= y >> 11
        y ^= (y << 7) & 0x9D2C5680
        y ^= (y << 15) & 0xEFC60000
        y ^= y >> 18

        return y & 0xFFFFFFFF  # Return 32-bit unsigned integer

    def genrand_int31(self) -> int:
        """Generates a random number on [0, 0x7FFFFFFF]-interval."""
        return self.genrand_int32() >> 1

    def next_bytes(self, length: int) -> bytes:
        """Generates random bytes."""
        return b"".join(
            self.genrand_int31().to_bytes(4, "little", signed=False)
            for _ in range(0, length, 4)
        )[:length]


class UnityUtils:
    @staticmethod
    def search_unity_pack(
            pack_path: str,
            data_type: list | None = None,
            data_name: list | None = None,
            condition_connect: bool = False,
            read_obj_anyway: bool = False,
    ) -> list[ObjectReader] | None:
        data_list: list[ObjectReader] = []
        type_passed = False
        try:
            env = UnityPy.load(pack_path)
            for obj in env.objects:
                if data_type and obj.type.name in data_type:
                    if condition_connect:
                        type_passed = True
                    else:
                        data_list.append(obj)
                if read_obj_anyway or type_passed:
                    data = obj.read()
                    if data_name and data.m_Name in data_name:
                        if not (condition_connect or type_passed):
                            continue
                        data_list.append(obj)
        except:
            pass
        return data_list


def decode_server_url(data: bytes) -> str:
    ciphers = {
        "ServerInfoDataUrl": "X04YXBFqd3ZpTg9cKmpvdmpOElwnamB2eE4cXDZqc3ZgTg==",
        "DefaultConnectionGroup": "tSrfb7xhQRKEKtZvrmFjEp4q1G+0YUUSkirOb7NhTxKfKv1vqGFPEoQqym8=",
        "SkipTutorial": "8AOaQvLC5wj3A4RC78L4CNEDmEL6wvsI",
        "Language": "wL4EWsDv8QX5vgRaye/zBQ==",
    }
    b64_data = base64.b64encode(data).decode()
    json_str = convert_string(b64_data, create_key("GameMainConfig"))
    obj = json.loads(json_str)
    encrypted_url = obj[ciphers["ServerInfoDataUrl"]]
    url = convert_string(encrypted_url, create_key("ServerInfoDataUrl"))
    return url


if __name__ == "__main__":
    for f in os.listdir(os.getcwd()):
        if url_obj := UnityUtils.search_unity_pack(f, ["TextAsset"], ["GameMainConfig"], True):
            url = decode_server_url(url_obj[0].read().m_Script.encode("utf-8", "surrogateescape"))
            print(url)
            break
```

4. 脚本执行后打印: https://yostar-serverinfo.bluearchiveyostar.com/r79_56_swwin27e1u6czsvrxshi.json
5. 访问上一步打印的地址得到json如下, 其中`1.56`下的`AddressablesCatalogUrlRoot`对应的值便是最新的资源服务器的地址`https://prod-clientpatch.bluearchiveyostar.com/r79_zqy5uxhd13gi68vaatbd`

```json
{
  "ConnectionGroups": [
    {
      "Name": "Prod-Audit",
      "ManagementDataUrl": "https://prod-noticeindex.bluearchiveyostar.com/prod/index.json",
      "IsProductionAddressables": true,
      "ApiUrl": "https://prod-game.bluearchiveyostar.com:5000/api/",
      "GatewayUrl": "https://prod-gateway.bluearchiveyostar.com:5100/api/",
      "KibanaLogUrl": "https://prod-logcollector.bluearchiveyostar.com:5300",
      "ProhibitedWordBlackListUri": "https://prod-notice.bluearchiveyostar.com/prod/ProhibitedWord/blacklist.csv",
      "ProhibitedWordWhiteListUri": "https://prod-notice.bluearchiveyostar.com/prod/ProhibitedWord/whitelist.csv",
      "CustomerServiceUrl": "https://bluearchive.jp/contact-1-hint",
      "OverrideConnectionGroups": [
        {
          "Name": "1.0",
          "AddressablesCatalogUrlRoot": "https://prod-clientpatch.bluearchiveyostar.com/m28_1_0_1_mashiro3"
        },
        {
          "Name": "1.56",
          "AddressablesCatalogUrlRoot": "https://prod-clientpatch.bluearchiveyostar.com/r79_zqy5uxhd13gi68vaatbd"
        }
      ],
      "BundleVersion": "li3pmyogha"
    }
  ]
}
```


#### 参考
* [https://github.com/ZM-Kimu/Blue-Archive-Asset-Downloader](https://github.com/ZM-Kimu/Blue-Archive-Asset-Downloader)
* [https://github.com/Deathemonic/BA-AD](https://github.com/Deathemonic/BA-AD)
