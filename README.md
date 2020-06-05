# InfluxDB Arduino Client

Simple Arduino client for writing and reading data from [InfluxDB](https://www.influxdata.com/products/influxdb-overview/), it doesn't matter whether a local server or InfluxDB Cloud. Library supports authentication, secure communication over TLS, [batching](#writing-in-batches), [automatic retrying](#buffer-handling-and-retrying) on server backpressure and connection failure.

It also allows setting data in various formats, automatically escapes special characters and offers specifying timestamp in various precisions. 

Library support both [InfluxDB 2](#basic-code-for-influxdb-2) and [InfluxDB 1](#basic-code-for-influxdb-2).

This is a new implementation and API, [original API](#original-api) is still supported. 

Supported devices: ESP8266 (2.7+) and ESP32 (1.0.3+).

- [Basic code for InfluxDB 2](#basic-code-for-influxdb-2)
- [Basic code for InfluxDB 1](#basic-code-for-influxdb-1)
- [Connecting to InfluxDB Cloud 2](#connecting-to-influxdb-cloud-2)
- [Writing in Batches](#writing-in-batches)
  - [Timestamp](#timestamp)
  - [Configure Timel](#configure-time)
  - [Batch Size](#batch-size)
- [Buffer Handling and Retrying](#buffer-handling-and-retrying)
- [Write Options](#write-options)
- [Secure Connection](#secure-connection)
    - [InfluxDb 2](#influxdb-2)
    - [InfluxDb 1](#influxdb-1)
- [Querying](#querying)
- [Original API](#original-api)
- [Troubleshooting](#troubleshooting)
- [Contributing](#contributing)
- [License](#license)


## Basic code for InfluxDB 2
Using client is very easy. After [seting up InfluxDB 2 server](https://v2.docs.influxdata.com/v2.0/get-started), first define connection parameters and a client instance:
```cpp
// InfluxDB 2 server url, e.g. http://192.168.1.48:9999 (Use: InfluxDB UI -> Load Data -> Client Libraries)
#define INFLUXDB_URL "influxdb-url"
// InfluxDB 2 server or cloud API authentication token (Use: InfluxDB UI -> Load Data -> Tokens -> <select token>)
#define INFLUXDB_TOKEN "token"
// InfluxDB 2 organization name or id (Use: InfluxDB UI -> Settings -> Profile -> <name under tile> )
#define INFLUXDB_ORG "org"
// InfluxDB 2 bucket name (Use: InfluxDB UI -> Load Data -> Buckets)
#define INFLUXDB_BUCKET "bucket"

// Single InfluxDB instance
InfluxDBClient client(INFLUXDB_URL, INFLUXDB_ORG, INFLUXDB_BUCKET, INFLUXDB_TOKEN);
```

The next step is adding data. Single data row is represented by the `Point` class. It consists of measurement name (like a table name), tags (which labels data) and fields (values to store):
```cpp
// Define data point with measurement name 'device_status`
Point pointDevice("device_status");
// Set tags
pointDevice.addTag("device", "ESP8266");
pointDevice.addTag("SSID", WiFi.SSID());
// Add data
pointDevice.addField("rssi", WiFi.RSSI());
pointDevice.addField("uptime", millis());
```

And finally, write data to db:
```cpp
// Write data
client.writePoint(pointDevice);
```

Complete source code is available in [BasicWrite example](examples/BasicWrite/BasicWrite.ino).

Data can be seen in the InfluxDB UI immediately. Use [Data Explorer](https://v2.docs.influxdata.com/v2.0/visualize-data/explore-metrics/) or create a [Dashboard](https://v2.docs.influxdata.com/v2.0/visualize-data/dashboards/).

## Basic code for InfluxDB 1
Using InfluxDB Arduino client for InfluxDB 1 is almost the same as for InfluxDB 2. The only difference is that InfluxDB 1 uses _database_ as classic name for data storage instead of bucket and the server is unsecured by default.
There is just different `InfluxDBClient contructor` and  `setConnectionParametersV1` function for setting also security params. Everything else remains the same.

```cpp
// InfluxDB server url, e.g. http://192.168.1.48:8086 (don't use localhost, always server name or ip address)
#define INFLUXDB_URL "influxdb-url"
// InfluxDB database name 
#define INFLUXDB_DB_NAME "database"

// Single InfluxDB instance
InfluxDBClient client(INFLUXDB_URL, INFLUXDB_DB_NAME);

// Define data point with measurement name 'device_status`
Point pointDevice("device_status");
// Set tags
pointDevice.addTag("device", "ESP8266");
pointDevice.addTag("SSID", WiFi.SSID());
// Add data
pointDevice.addField("rssi", WiFi.RSSI());
pointDevice.addField("uptime", millis());

// Write data
client.writePoint(pointDevice);
```

Complete source code is available in [BasicWrite example](examples/BasicWrite/BasicWrite.ino)

## Connecting to InfluxDB Cloud 2
Instead of setting up local InfluxDB 2 server, it is possible to quickly [start with InfluxDB Cloud 2](https://v2.docs.influxdata.com/v2.0/cloud/get-started/) with [Free Plan](https://v2.docs.influxdata.com/v2.0/cloud/pricing-plans/#free-plan).

Connecting Arduino client to InfuxDB Cloud server requires few additional steps.
InfluxDBCloud uses secure communication (https) and we need to tell the client to trust this connection.
Connection parameters are almost the same as above, the only difference is that server URL now points to the InfluxDB Cloud 2, where you've got after you've finished creating InfluxDB Cloud 2 subscription. You will find correct server URL in  `InfluxDB UI -> Load Data -> Client Libraries`.
```cpp
//Include also InfluxClould 2 CA certificate
#include <InfluxCloud.h>
// InfluxDB 2 server or cloud url, e.g. https://eu-central-1-1.aws.cloud2.influxdata.com (Use: InfluxDB UI -> Load Data -> Client Libraries)
#define INFLUXDB_URL "influxdb-url"
// InfluxDB 2 server or cloud API authentication token (Use: InfluxDB UI -> Load Data -> Tokens -> <select token>)
#define INFLUXDB_TOKEN "token"
// InfluxDB 2 organization name or id (Use: InfluxDB UI -> Settings -> Profile -> <name under tile> )
#define INFLUXDB_ORG "org"
// InfluxDB 2 bucket name (Use: InfluxDB UI -> Load Data -> Buckets)
#define INFLUXDB_BUCKET "bucket"
```

You need to pass an additional parameter to the client constructor, which is a certificate of the server to trust. Constant `InfluxDbCloud2CACert` contains the InfluxDB Cloud 2 CA certificate, which is predefined in this library: 
```cpp
// Single InfluxDB instance
InfluxDBClient client(INFLUXDB_URL, INFLUXDB_ORG, INFLUXDB_BUCKET, INFLUXDB_TOKEN, InfluxDbCloud2CACert);
```
Read more about [secure connection](#secure-connection).

Additionally, time needs to be synced:
```cpp
// Synchronize time with NTP servers and set timezone
// Accurate time is necessary for certificate validaton and writing in batches
// For the fastest time sync find NTP servers in your area: https://www.pool.ntp.org/zone/
configTzTime(TZ_INFO "pool.ntp.org", "time.nis.gov");
```
Read more about time synchronization in [Configure Time](#configure-time).

Defining data and writing it to the DB is the same as in the case of [BasicWrite](#basic-code):
```cpp
// Define data point with measurement name 'device_status`
Point pointDevice("device_status");
// Set tags
pointDevice.addTag("device", "ESP8266");
pointDevice.addTag("SSID", WiFi.SSID());
// Add data
pointDevice.addField("rssi", WiFi.RSSI());
pointDevice.addField("uptime", millis());

// Write data
client.writePoint(pointDevice);
```
Complete source code is available in [SecureWrite example](examples/SecureWrite/SecureWrite.ino).

## Writing in Batches
InfluxDB client for Arduino can write data in batches. A batch is simply a set of points that will be sent at once. To create a batch, the client will keep all points until the number of points reaches the batch size and then it will write all points at once to the InfluDB server. This is often more efficient than writing each point separately. 

### Timestamp
If using batch writes, the timestamp should be employed. Timestamp specifies the time where data was gathered and it is used in the form of a number of seconds (milliseconds, etc) from epoch (1.1.1970) UTC.
If points have no timestamp assigned, InfluxDB assigns timestamp at the time of writing, which could happen much later than the data has been obtained, because final batch write will happen when the batch is full (or when [flush buffer](#buffer-handling-and-retrying) is forced).

InfuxDB allows sending timestamp in various precisions - nanoseconds, microseconds, milliseconds or seconds. The milliseconds precision is usually enough for using on Arduino.

The client has to be configured with time precision. The default settings is not using the timestamp. The `setWriteOptions` functions allow setting various parameters and one of them is __write precision__:
``` cpp
// Set write precision to milliseconds. Leave other parameters default.
client.setWriteOptions(WritePrecision::MS);
```
When a write precision is configured, the client will automatically assign current time to the timestamp of each written point, which doesn't have a timestamp assigned. 

If you want to manage timestamp on your own, there are several ways how to set timestamp explicitly.
- `setTime(WritePrecision writePrecision)` - Sets timestamp to actual time in desired precision
- `setTime(unsigned long seconds)` -  Sets timestamp in seconds since epoch. Write precision must be set to `S` 
- `setTime(String timestamp)` - Set custom timestamp in precision specified in InfluxDBClient. 


### Configure Time
Dealing with timestamps, and also validating server or CA certificate, requires the device has correctly set time. This can be done with just one line of code:
```cpp
// Synchronize time with NTP servers and set timezone
// Accurate time is necessary for certificate validaton and writing in batches
// For the fastest time sync find NTP servers in your area: https://www.pool.ntp.org/zone/
configTzTime("PST8PDT", "pool.ntp.org", "time.nis.gov");
```
The `configTzTime` function starts the time synchronization with NTP servers. The first parameter specifies timezone information, which is important for distinguishing UTC and a local timezone and for daylight saving changes.
The last two string parameters are the internet addresses of NTP servers. Check [pool.ntp.org](https://www.pool.ntp.org/zone) for address of some local NTP servers.

Timezone string details are described at [https://www.gnu.org/software/libc/manual/html_node/TZ-Variable.html](https://www.gnu.org/software/libc/manual/html_node/TZ-Variable.html).
Values for some timezones:
- Central Europe: `CET-1CEST,M3.5.0,M10.5.0/3`
- Eastern: `EST5EDT`
- Japanesse: `JST-9`
- Pacific Time: `PST8PDT`

There is also another function for syncing time, which takes timezone and DST offset. As DST info is set via static offset it will create local time problem when DST change will occur. 
It's declaration is following:
```cpp
configTime(long gmtOffset_sec, int daylightOffset_sec, const char* server1, const char* server2 = nullptr, const char* server3 = nullptr);
```

In the example code it would be:
```cpp
// Synchronize time with NTP servers
// Accurate time is necessary for certificate validaton and writing in batches
configTime(3600, 3600, "pool.ntp.org", "time.nis.gov");
```

Both `configTzTime` and `configTime` functions are asynchronous. This means that calling the functions just starts the time synchronization. Time is often not synchronized yet upon returning from call.

There is a helper function `timeSync` provided with the this library. The function starts time synchronization by calling the `configTzTime` and waits maximum 20 seconds for time is synchronized. It prints progress info and final local time to the `Serial`. 
`timeSync` has the same signature and `configTzTime` and it is included with the main header file `InfluxDbClient.h`:
```cpp
// Synchronize time with NTP servers and waits for completition. Prints waiting progress and final synchronized time to the Serial.
// Accurate time is necessary for certificate validion and writing points in batch
// For the fastest time sync find NTP servers in your area: https://www.pool.ntp.org/zone/
void timeSync(const char *tzInfo, const char* ntpServer1, const char* ntpServer2 = nullptr, const char* ntpServer3 = nullptr);
```

### Batch Size
Setting batch size depends on data gathering and DB updating strategy.

If data is written in short periods (seconds), batch size should be according to expected write periods and update frequency requirements. 
For example, if you would like to see updates (on the dashboard or in processing) each minute and you are measuring single data (1 point) each 10s (6 points per minute), batch size should be 6. In case it is enough to update each hour and you are creating 1 point at once each minute, your batch size should be 60. The maximum recommended batch size is 200. It depends on the RAM of the device (80KB for ESP8266 and 512KB for ESP32).

In case that data should be written in longer periods and gathered data consists of several points batch size should be set to an expected number of points.

To set batch size we use [setWriteOptions](#write-options) function, where second parameter controls batch size:
```cpp
// Enable messages batching
client.setWriteOptions(WritePrecision::MS, 10);
```
Writing point will add a point to the underlying buffer until the batch size is reached:
```cpp
// Write first point to the buffer
client.writePoint(point1);
// Write second point to the buffer
client.writePoint(point2);
..
// Write nineth point to the buffer, returns 
client.writePoint(point9);
// Writing tenth point will cause flushing buffer
if(!client.writePoint(point10)) {
    Serial.print("InfluxDB write failed: ");
    Serial.println(client.getLastErrorMessage());
}
```

In case of a number of points is not always the same, set batch size to the maximum number of points and use the `flushBuffer()` function to force writing to DB. See [Buffer Handling](#buffer-handling-and-retrying) for more details.

## Buffer Handling and Retrying
InfluxDB contains an underlying buffer for handling writing in batches and automatic retrying on server backpressure and connection failure.

Its size is controled by the 3rd parameter of [setWriteOptions](#write-options) function:
```cpp
// Enable messages batching
client.setWriteOptions(WritePrecision::MS, 10, 30);
```
The third parameter specifies the buffer size. The recommended size is at least 2 x batch size. 

State of the buffer can be determined via two functions:
 - `isBufferEmpty()` - Returns true if buffer is empty
 - `isBufferFull()` - Returns true if buffer is full
 
 Full buffer can occur when there is a problem with an internet connection or the InfluxDB server is overloaded. In such cases, points to write remains in buffer. When more points are added and connection problem remains, the buffer will reach the top and new points will overwrite older points.

 Each attempt to write a point will try to send older points in the buffer. So, the `isBufferFull()` function can be used to skip low priority points.

The `flushBuffer()` function can be used to force writing, even the number of points in the buffer is lower than the batch size. With the help of the `isBufferEmpty()` function a check can be made before a device goes to sleep:

 ```cpp
  // Check whether buffer in not empty
  if (!client.isBufferEmpty()) {
      // Write all remaining points to db
      client.flushBuffer();
  }
```

Other functions for dealing with buffer:
 - `checkBuffer()` - Checks point buffer status and flushes if the number of points reaches batch size or flush interval runs out. This main function for controlling buffer and it is used internally.
 - `resetBuffer()` - Clears the buffer.

Check [SecureBatchWrite example](examples/SecureBatchWrite/SecureBatchWrite.ino) for example code of buffer handling functions.

## Write Options
Writing points can be controlled via several parameters in `setWriteOptions` function:

| Parameter | Default Value | Meaning |
|-----------|---------------|---------| 
| precision | `WritePrecision::NoTime` | Timestamp precision of written data |
| batchSize | `1` | Number of points that will be written to the database at once |
| bufferSize | `5` | Maximum number of points in buffer. Buffer contains new data that will be written to the database and also data that failed to be written due to network failure or server overloading |
| flushInterval | `60` | Maximum time(in seconds) data will be held in buffer before are written to the db |
| preserveConnection | `false` | true if underlying HTTP connection should be kept open |

## Secure Connection
Connecting to a secured server requires configuring client to trust the server. This is achieved by providing client with a server certificate, certificate authority certificate or certificate SHA1 fingerprint. 

Note: `HTTPClient` in the current ESP32 arduino SDK (1.0.4) doesn't validate server certificate, so providing server certificate is not necessary. But it is definitely safer to do it, as it can change in the future.
Other limitation of ESP32 arduino SDK (1.0.4) is that `WiFiClientSecure` doesn't support fingerprint to validate server certificate.

Certificate (in PEM format) or SHA1 fingerprint can be placed in flash memory to save initial RAM:
```cpp
// Certificate of Certificate Authority of InfluxData Cloud 2 servers
const char InfluxDbCloud2CACert[] PROGMEM =  R"EOF( 
-----BEGIN CERTIFICATE-----
MIIGEzCCA/ugAwIBAgIQfVtRJrR2uhHbdBYLvFMNpzANBgkqhkiG9w0BAQwFADCB
iDELMAkGA1UEBhMCVVMxEzARBgNVBAgTCk5ldyBKZXJzZXkxFDASBgNVBAcTC0pl
cnNleSBDaXR5MR4wHAYDVQQKExVUaGUgVVNFUlRSVVNUIE5ldHdvcmsxLjAsBgNV
BAMTJVVTRVJUcnVzdCBSU0EgQ2VydGlmaWNhdGlvbiBBdXRob3JpdHkwHhcNMTgx
MTAyMDAwMDAwWhcNMzAxMjMxMjM1OTU5WjCBjzELMAkGA1UEBhMCR0IxGzAZBgNV
BAgTEkdyZWF0ZXIgTWFuY2hlc3RlcjEQMA4GA1UEBxMHU2FsZm9yZDEYMBYGA1UE
ChMPU2VjdGlnbyBMaW1pdGVkMTcwNQYDVQQDEy5TZWN0aWdvIFJTQSBEb21haW4g
VmFsaWRhdGlvbiBTZWN1cmUgU2VydmVyIENBMIIBIjANBgkqhkiG9w0BAQEFAAOC
AQ8AMIIBCgKCAQEA1nMz1tc8INAA0hdFuNY+B6I/x0HuMjDJsGz99J/LEpgPLT+N
TQEMgg8Xf2Iu6bhIefsWg06t1zIlk7cHv7lQP6lMw0Aq6Tn/2YHKHxYyQdqAJrkj
eocgHuP/IJo8lURvh3UGkEC0MpMWCRAIIz7S3YcPb11RFGoKacVPAXJpz9OTTG0E
oKMbgn6xmrntxZ7FN3ifmgg0+1YuWMQJDgZkW7w33PGfKGioVrCSo1yfu4iYCBsk
Haswha6vsC6eep3BwEIc4gLw6uBK0u+QDrTBQBbwb4VCSmT3pDCg/r8uoydajotY
uK3DGReEY+1vVv2Dy2A0xHS+5p3b4eTlygxfFQIDAQABo4IBbjCCAWowHwYDVR0j
BBgwFoAUU3m/WqorSs9UgOHYm8Cd8rIDZsswHQYDVR0OBBYEFI2MXsRUrYrhd+mb
+ZsF4bgBjWHhMA4GA1UdDwEB/wQEAwIBhjASBgNVHRMBAf8ECDAGAQH/AgEAMB0G
A1UdJQQWMBQGCCsGAQUFBwMBBggrBgEFBQcDAjAbBgNVHSAEFDASMAYGBFUdIAAw
CAYGZ4EMAQIBMFAGA1UdHwRJMEcwRaBDoEGGP2h0dHA6Ly9jcmwudXNlcnRydXN0
LmNvbS9VU0VSVHJ1c3RSU0FDZXJ0aWZpY2F0aW9uQXV0aG9yaXR5LmNybDB2Bggr
BgEFBQcBAQRqMGgwPwYIKwYBBQUHMAKGM2h0dHA6Ly9jcnQudXNlcnRydXN0LmNv
bS9VU0VSVHJ1c3RSU0FBZGRUcnVzdENBLmNydDAlBggrBgEFBQcwAYYZaHR0cDov
L29jc3AudXNlcnRydXN0LmNvbTANBgkqhkiG9w0BAQwFAAOCAgEAMr9hvQ5Iw0/H
ukdN+Jx4GQHcEx2Ab/zDcLRSmjEzmldS+zGea6TvVKqJjUAXaPgREHzSyrHxVYbH
7rM2kYb2OVG/Rr8PoLq0935JxCo2F57kaDl6r5ROVm+yezu/Coa9zcV3HAO4OLGi
H19+24rcRki2aArPsrW04jTkZ6k4Zgle0rj8nSg6F0AnwnJOKf0hPHzPE/uWLMUx
RP0T7dWbqWlod3zu4f+k+TY4CFM5ooQ0nBnzvg6s1SQ36yOoeNDT5++SR2RiOSLv
xvcRviKFxmZEJCaOEDKNyJOuB56DPi/Z+fVGjmO+wea03KbNIaiGCpXZLoUmGv38
sbZXQm2V0TP2ORQGgkE49Y9Y3IBbpNV9lXj9p5v//cWoaasm56ekBYdbqbe4oyAL
l6lFhd2zi+WJN44pDfwGF/Y4QA5C5BIG+3vzxhFoYt/jmPQT2BVPi7Fp2RBgvGQq
6jG35LWjOhSbJuMLe/0CjraZwTiXWTb2qHSihrZe68Zk6s+go/lunrotEbaGmAhY
LcmsJWTyXnW0OMGuf1pGg+pRyrbxmRE1a6Vqe8YAsOf4vmSyrcjC8azjUeqkk+B5
yOGBQMkKW+ESPMFgKuOXwIlCypTPRpgSabuY0MLTDXJLR27lk8QyKGOHQ+SwMj4K
00u/I5sUKUErmgQfky3xxzlIPK1aEn8=
-----END CERTIFICATE-----
)EOF";

// Fingerprint of Certificate Authority of InfluxData Cloud 2 servers
const char InfluxDbCloud2CAFingerprint[] PROGMEM = "9B:62:0A:63:8B:B1:D2:CA:5E:DF:42:6E:A3:EE:1F:19:36:48:71:1F";
```

### InfluxDb 2
There are two ways to set certificate or fingerprint to trust a server:
 - Use full param constructor
```cpp
// InfluxDB client instance with preconfigured InfluxCloud certificate
InfluxDBClient client(INFLUXDB_URL, INFLUXDB_ORG, INFLUXDB_BUCKET, INFLUXDB_TOKEN, InfluxDbCloud2CACert);
```
- Use `setConnectionParams` function:
```cpp
// InfluxDB client instance 
InfluxDBClient client;

void setup() {
    // configure client
    client.setConnectionParams(INFLUXDB_URL, INFLUXDB_ORG, INFLUXDB_BUCKET, INFLUXDB_TOKEN, InfluxDbCloud2CACert);
}
```
### InfluxDb 1

Use `setConnectionParamsV1` function:
```cpp
// InfluxDB client instance 
InfluxDBClient client;

void setup() {
    // configure client
    client.setConnectionParamsV1(INFLUXDB_URL, INFLUXDB_DATABASE, INFLUXDB_USER, INFLUXDB_PASSWORD, InfluxDbCloud2CACert);
}
```
Another important prerequisity to sucessfully validate server or CA certificate is to have properly synchronized time. More on this in [Configure Timel](#configure-time).

Note: Time synchronization is not required for validating server certificate via SHA1 fingerprint.

## Querying
InfluxDB 2 and InfluxDB 1.7+ (with [enabled flux](https://docs.influxdata.com/influxdb/latest/administration/config/#flux-enabled-false)) uses [Flux](https://www.influxdata.com/products/flux/) to process and query data. InfluxDB client for Arduino offers a simple, but powerful, way how to query data with `query` function. It parses response line by line, so it can read a huge responses (thousands data lines), without consuming a lot device memory.

The `query` returns `FluxQueryResult` object, which parses response and provides useful getters for accessing values from result set. 

InfluxDB flux query result set is returned in the CSV format. In the example bellow, the first line contains type information and the second columns name and the rest is data:
```CSV
#datatype,string,long,dateTime:RFC3339,dateTime:RFC3339,dateTime:RFC3339,long,string,string,string,string
,result,table,_start,_stop,_time,_value,SSID,_field,_measurement,device
,_result,0,2020-05-18T15:06:00.475253281Z,2020-05-19T15:06:00.475253281Z,2020-05-19T13:07:13Z,-55,667G,rssi,wifi_status,ESP32
,_result,0,2020-05-18T15:06:00.475253281Z,2020-05-19T15:06:00.475253281Z,2020-05-19T13:07:27Z,-54,667G,rssi,wifi_status,ESP32
,_result,0,2020-05-18T15:06:00.475253281Z,2020-05-19T15:06:00.475253281Z,2020-05-19T13:07:40Z,-54,667G,rssi,wifi_status,ESP32
,_result,0,2020-05-18T15:06:00.475253281Z,2020-05-19T15:06:00.475253281Z,2020-05-19T13:07:54Z,-54,667G,rssi,wifi_status,ESP32
,_result,0,2020-05-18T15:06:00.475253281Z,2020-05-19T15:06:00.475253281Z,2020-05-19T13:08:07Z,-55,667G,rssi,wifi_status,ESP32
,_result,0,2020-05-18T15:06:00.475253281Z,2020-05-19T15:06:00.475253281Z,2020-05-19T13:08:20Z,-56,667G,rssi,wifi_status,ESP32
```

Accessing data using `FluxQueryResult` requires knowing the query result structure, especially the name and the type of the column. The best practise is to tune query
in the `InfluxDB Data Explorer` and use the final query with this library.

 Browsing thought the result set is done by repeatedly calling the `next()` method, until it returns false. Unsuccesful reading is distinqushed by non empty value from the `getError()` method.
 As a flux query result can contain several tables, differing by grouping key, use the `hasTableChanged()` method to know when there is a new table.
 Single values are returned using the `getValueByIndex()` or `getValueByName()` methods.
 All row values at once are retreived by the `getValues()` method.
  Always call the `close()` method at the of reading.

A value in the flux query result column, retrieved by the `getValueByIndex()` or `getValueByName()` methods, is represented by the `FluxValue` object.
It provides getter methods for supported flux types:

| Flux type | Getter | C type |
| ----- | ------ |  --- |
| long | getLong() | long |
| unsignedLong | getUnsignedLong() | unsingned long |
| dateTime:RFC3339, dateTime:RFC3339Nano | getDateTime() |  [FluxDateTime](src/query/FluxTypes.h#L100) |
| bool | getBool() | bool |
| double | bool | double |
| string, base64binary, duration | getString() | String |

Calling improper type getter will result in a zero (empty) value. 

Check for null (missing) value usig the `isNull()` method.  

Use the `getRawValue()` method for getting original string form.

```cpp
// Construct a Flux query
// Query will find RSSI for last 24 hours for each connected WiFi network with this device computed by given selector function
String query = "from(bucket: \"my-bucket\") |> range(start: -24h) |> filter(fn: (r) => r._measurement == \"wifi_status\" and r._field == \"rssi\"";
query += "and r.device == \"ESP32\")";
query += "|> max()";

// Send query to the server and get result
FluxQueryResult result = client.query(query);

// Iterate over rows. Even there is just one row, next() must be called at least once.
while (result.next()) {
  // Get typed value for flux result column 'SSID'
  String ssid = result.getValueByName("SSID").getString();
  Serial.print("SSID '");
  Serial.print(ssid);

  Serial.print("' with RSSI ");

  // Get converted value for flux result column '_value' where there is RSSI value
  long value = result.getValueByName("_value").getLong();
  Serial.print(value);

  // Format date-time for printing
  // Format string according to http://www.cplusplus.com/reference/ctime/strftime/
  String timeStr = time.format("%F %T");

  Serial.print(" at ");
  Serial.print(timeStr);
  
  Serial.println();
}

// Check if there was an error
if(result.getError() != "") {
  Serial.print("Query result error: ");
  Serial.println(result.getError());
}
``` 
Complete source code is available in [QueryAggregated example](examples/QueryAggregated/QueryAggregated.ino).

## Original API

### Initialization
```cpp
 #define INFLUXDB_HOST "192.168.0.32"
 #define INFLUXDB_PORT 1337
 #define INFLUXDB_DATABASE "test"
 //if used with authentication
 #define INFLUXDB_USER "user"
 #define INFLUXDB_PASS "password"

 // connect to WiFi

 Influxdb influx(INFLUXDB_HOST); // port defaults to 8086, use 9999 for v2
 // or to use a custom port
 Influxdb influx(INFLUXDB_HOST, INFLUXDB_PORT);

 // set the target database
 influx.setDb(INFLUXDB_DATABASE);
 // or use a db with auth
 influx.setDbAuth(INFLUXDB_DATABASE, INFLUXDB_USER, INFLUXDB_PASS) // with authentication

// To use the v2.0 InfluxDB
influx.setVersion(2);
influx.setOrg("myOrganization");
influx.setBucket("myBucket");
influx.setToken("myToken");
influx.setPort(9999);
```

### Sending a single measurement
**Using an InfluxData object:**
```cpp
// create a measurement object
InfluxData measurement ("temperature");
measurement.addTag("device", d2);
measurement.addTag("sensor", "dht11");
measurement.addValue("value", 24.0);

// write it into db
influx.write(measurement);
```

**Using raw-data**
```cpp
 influx.write("temperature,device=d2,sensor=dht11 value=24.0")
```

### Write multiple data points at once
Batching measurements and send them with a single request will result in a much higher performance.
```cpp

InfluxData measurement1 = readTemperature()
influx.prepare(measurement1)

InfluxData measurement2 = readLight()
influx.prepare(measurement2)

InfluxData measurement3 = readVoltage()
influx.prepare(measurement3)

// writes all prepared measurements with a single request into db.
boolean success = influx.write();
```

## Troubleshooting
All db methods return status. Value `false` means something went wrong. Call `getLastErrorMessage()` to get the error message.

When error message doesn't help to explain the bad behavior, go to the library sources and in the file `src/InfluxDBClient.cpp` uncomment line 44:
```cpp
// Uncomment bellow in case of a problem and rebuild sketch
#define INFLUXDB_CLIENT_DEBUG
```
Then upload your sketch again and see the debug output in the Serial Monitor.

If you couldn't solve a problem by yourself, please, post an issue including the debug output.

## Contributing

If you would like to contribute code you can do through GitHub by forking the repository and sending a pull request into the `master` branch.

## License

The InfluxDB Arduino Client is released under the [MIT License](https://opensource.org/licenses/MIT).
