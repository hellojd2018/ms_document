# Eureka REST操作

以下是可供非Java应用程序使用Eureka 的REST操作。

**appID**是应用程序的名称，**instanceID**是与**实例**关联的唯一ID。在AWS云中，instanceID是**实例的实例ID**，在其他数据中心中，它是实例的**主机名**。

对于JSON / XML，提供的内容类型必须是**application / xml**或**application / json**。

| **手术**                                                     | **HTTP动作**                                                 | **描述**                                                     |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Register new application instance                            | POST / eureka / v2 / apps / **appID**                        | 输入：JSON / XML有效负载HTTP代码：204成功                    |
| De-register application instance                             | DELETE / eureka / v2 / apps / **appID** / **instanceID**     | HTTP代码：成功200                                            |
| Send application instance heartbeat                          | PUT / eureka / v2 / apps / **appID** / **instanceID**        | HTTP代码： * 200成功 * 404如果**instanceID**不存在           |
| Query for all instances                                      | GET / eureka / v2 / apps                                     | HTTP代码：成功时输出200输出：JSON / XML                      |
| Query for all **appID** instances                            | GET / eureka / v2 / apps / **appID**                         | HTTP代码：成功时输出200输出：JSON / XML                      |
| 查询特定的**appID** /**instanceID**                          | GET / eureka / v2 / apps / **appID** / **instanceID**        | HTTP代码：成功时输出200输出：JSON / XML                      |
| 查询特定的**instanceID**                                     | GET / eureka / v2 / instances / **instanceID**               | HTTP代码：成功时输出200输出：JSON / XML                      |
| Query for a specific **appID**/**instanceID**                | PUT / eureka / v2 / apps / **appID** / **instanceID** / status？value = OUT_OF_SERVICE | HTTP代码： * 200成功 * 500失败                               |
| Move instance back into service (remove override)            | DELETE / eureka / v2 / apps / **appID** / **instanceID** / status？value = UP（值= UP是可选的，由于删除了覆盖，它被用作回退状态的建议） | HTTP代码： * 200成功 * 500失败                               |
| Update metadata                                              | PUT / eureka / v2 / apps / **appID** / **instanceID** / metadata？key = value | HTTP代码： * 200成功 * 500失败                               |
| Query for all instances under a particular **vip address**   | GET / eureka / v2 / vips / **vipAddress**                    | * HTTP代码：200成功输出：JSON /XML  * 404如果**vipAddress**不存在。 |
| Query for all instances under a particular **secure vip address** | GET / eureka / v2 / svips / **svipAddress**                  | * HTTP代码：200成功输出：JSON /XML  * 404如果**svipAddress**不存在。 |

**寄存器**

注册时，您需要发布符合此XSD的XML（或JSON）主体：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<xsd:schema xmlns:xsd="http://www.w3.org/2001/XMLSchema" elementFormDefault="qualified" attributeFormDefault="unqualified">
    <xsd:element name="instance">
        <xsd:complexType>
            <xsd:all>
                <!-- hostName in ec2 should be the public dns name, within ec2 public dns name will
                     always resolve to its private IP -->
                <xsd:element name="hostName" type="xsd:string" />
                <xsd:element name="app" type="xsd:string" />
                <xsd:element name="ipAddr" type="xsd:string" />
                <xsd:element name="vipAddress" type="xsd:string" />
                <xsd:element name="secureVipAddress" type="xsd:string" />
                <xsd:element name="status" type="statusType" />
                <xsd:element name="port" type="xsd:positiveInteger" minOccurs="0" />
                <xsd:element name="securePort" type="xsd:positiveInteger" />
                <xsd:element name="homePageUrl" type="xsd:string" />
                <xsd:element name="statusPageUrl" type="xsd:string" />
                <xsd:element name="healthCheckUrl" type="xsd:string" />
               <xsd:element ref="dataCenterInfo" minOccurs="1" maxOccurs="1" />
                <!-- optional -->
                <xsd:element ref="leaseInfo" minOccurs="0"/>
                <!-- optional app specific metadata -->
                <xsd:element name="metadata" type="appMetadataType" minOccurs="0" />
            </xsd:all>
        </xsd:complexType>
    </xsd:element>

    <xsd:element name="dataCenterInfo">
        <xsd:complexType>
             <xsd:all>
                 <xsd:element name="name" type="dcNameType" />
                 <!-- metadata is only required if name is Amazon -->
                 <xsd:element name="metadata" type="amazonMetdataType" minOccurs="0"/>
             </xsd:all>
        </xsd:complexType>
    </xsd:element>

    <xsd:element name="leaseInfo">
        <xsd:complexType>
            <xsd:all>
                <!-- (optional) if you want to change the length of lease - default if 90 secs -->
                <xsd:element name="evictionDurationInSecs" minOccurs="0"  type="xsd:positiveInteger"/>
            </xsd:all>
        </xsd:complexType>
    </xsd:element>

    <xsd:simpleType name="dcNameType">
        <!-- Restricting the values to a set of value using 'enumeration' -->
        <xsd:restriction base = "xsd:string">
            <xsd:enumeration value = "MyOwn"/>
            <xsd:enumeration value = "Amazon"/>
        </xsd:restriction>
    </xsd:simpleType>

    <xsd:simpleType name="statusType">
        <!-- Restricting the values to a set of value using 'enumeration' -->
        <xsd:restriction base = "xsd:string">
            <xsd:enumeration value = "UP"/>
            <xsd:enumeration value = "DOWN"/>
            <xsd:enumeration value = "STARTING"/>
            <xsd:enumeration value = "OUT_OF_SERVICE"/>
            <xsd:enumeration value = "UNKNOWN"/>
        </xsd:restriction>
    </xsd:simpleType>

    <xsd:complexType name="amazonMetdataType">
        <!-- From <a class="jive-link-external-small" href="http://docs.amazonwebservices.com/AWSEC2/latest/DeveloperGuide/index.html?AESDG-chapter-instancedata.html" target="_blank">http://docs.amazonwebservices.com/AWSEC2/latest/DeveloperGuide/index.html?AESDG-chapter-instancedata.html</a> -->
        <xsd:all>
            <xsd:element name="ami-launch-index" type="xsd:string" />
            <xsd:element name="local-hostname" type="xsd:string" />
            <xsd:element name="availability-zone" type="xsd:string" />
            <xsd:element name="instance-id" type="xsd:string" />
            <xsd:element name="public-ipv4" type="xsd:string" />
            <xsd:element name="public-hostname" type="xsd:string" />
            <xsd:element name="ami-manifest-path" type="xsd:string" />
            <xsd:element name="local-ipv4" type="xsd:string" />
            <xsd:element name="hostname" type="xsd:string"/>       
            <xsd:element name="ami-id" type="xsd:string" />
            <xsd:element name="instance-type" type="xsd:string" />
        </xsd:all>
    </xsd:complexType>

    <xsd:complexType name="appMetadataType">
        <xsd:sequence>
            <!-- this is optional application specific name, value metadata -->
            <xsd:any minOccurs="0" maxOccurs="unbounded" processContents="skip"/>
        </xsd:sequence>
    </xsd:complexType>

</xsd:schema>
```

**更新**

```sh
Example : PUT /eureka/v2/apps/MYAPP/i-6589ef6

Response:
Status: 200 (on success)
404 (eureka doesn’t know about you, Register yourself first)
500 (failure)

```





**取消**

（如果Eureka没有从evictionDurationInSecs中的服务节点获取心跳，则该节点将自动取消注册）

```sh
Example : DELETE /eureka/v2/apps/MYAPP/i-6589ef6

Response:
Status: 200 (on success)
500 (failure)
```

