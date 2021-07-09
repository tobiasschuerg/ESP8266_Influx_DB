# InfluxDB Arduino Client

Simple Arduino client for writing and reading data from [InfluxDB](https://www.influxdata.com/products/influxdb-overview/), no matter whether it is a local server or InfluxDB Cloud. The library supports authentication, secure communication over TLS, [batching](#writing-in-batches), [automatic retrying](#buffer-handling-and-retrying) on server back-pressure and connection failure.

It also allows setting data in various formats, automatically escapes special characters and offers specifying timestamp in various precisions.

Library supports both [InfluxDB 2](#basic-code-for-influxdb-2) and [InfluxDB 1](#basic-code-for-influxdb-2).

This is a new implementation and the API, [original API](#original-api) is still supported.

Supported devices: ESP8266 (2.7+) and ESP32 (1.0.3+).

- [InfluxDB Arduino Client](#influxdb-arduino-client)
  - [Basic code for InfluxDB 2](#basic-code-for-influxdb-2)
  - [Basic code for InfluxDB 1](#basic-code-for-influxdb-1)
  - [Connecting to InfluxDB Cloud 2](#connecting-to-influxdb-cloud-2)
  - [Writing in Batches](#writing-in-batches)
    - [Timestamp](#timestamp)
    - [Configure Time](#configure-time)
    - [Batch Size](#batch-size)
  - [Buffer Handling and Retrying](#buffer-handling-and-retrying)
  - [Write Options](#write-options)
  - [HTTP Options](#http-options)
  - [Secure Connection](#secure-connection)
    - [InfluxDb 2](#influxdb-2)
    - [InfluxDb 1](#influxdb-1)
    - [Skipping certificate validation](#skipping-certificate-validation)
  - [Querying](#querying)
  - [Original API](#original-api)
    - [Initialization](#initialization)
    - [Sending a single measurement](#sending-a-single-measurement)
    - [Write multiple data points at once](#write-multiple-data-points-at-once)
  - [Troubleshooting](#troubleshooting)
  - [Contributing](#contributing)
  - [License](#license)


## Basic code for InfluxDB 2
After [setting up an InfluxDB 2 server](https://docs.influxdata.com/influxdb/v2.0/get-started/), first define connection parameters and a client instance:
```cpp
// InfluxDB 2 server url, e.g. http://192.168.1.48:8086 (Use: InfluxDB UI -> Load Data -> Client Libraries)
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

The next step is adding data. A single row of data is represented by the `Point` class. It consists of a measurement name (like a table name), tags (which labels data) and fields ( the values to store):
```cpp
// Define data point in the measurement named 'device_status`
Point pointDevice("device_status");
// Set tags
pointDevice.addTag("device", "ESP8266");
pointDevice.addTag("SSID", WiFi.SSID());
// Add data fields
pointDevice.addField("rssi", WiFi.RSSI());
pointDevice.addField("uptime", millis());
```

And finally, write the data to the database:
```cpp
// Write data
client.writePoint(pointDevice);
```

Complete source code is available in the [BasicWrite example](examples/BasicWrite/BasicWrite.ino).

Data can be seen in the InfluxDB UI immediately. Use the [Data Explorer](https://docs.influxdata.com/influxdb/v2.0/query-data/execute-queries/data-explorer/) or create a [Dashboard](https://docs.influxdata.com/influxdb/v2.0/visualize-data/dashboards/).

## Basic code for InfluxDB 1
Using InfluxDB Arduino client for InfluxDB 1 is almost the same as for InfluxDB 2. The only difference is that InfluxDB 1 uses _database_ as classic name for data storage instead of bucket and the server is unsecured by default.
There is also a different `InfluxDBClient constructor` and  `setConnectionParametersV1` function for setting the security params. Everything else remains the same.

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
Instead of setting up a local InfluxDB 2 server, it is possible to quickly [start with InfluxDB Cloud 2](https://docs.influxdata.com/influxdb/cloud/get-started/) with a [Free Plan](https://docs.influxdata.com/influxdb/cloud/account-management/pricing-plans/#free-plan).

InfluxDB Cloud uses secure communication over TLS (https). We need to tell the client to trust this connection. The paragraph bellow describes how to set trusted connection. However, InfluxDB cloud servers have only 3 months validity period. Their CA certificate, included in this library, is valid until 2035. Check [Skipping certification validation](#skipping-certificate-validation) for more details. 

Connecting an Arduino client to InfluxDB Cloud server requires a few additional steps comparing to connecting to local server.

Connection parameters are almost the same as above, the only difference is that server URL now points to the InfluxDB Cloud 2, you set up after you've finished creating an InfluxDB Cloud 2 subscription. You will find the correct server URL in  `InfluxDB UI -> Load Data -> Client Libraries`.
```cpp
//Include also InfluxCloud 2 CA certificate
#include <InfluxDbCloud.h>
// InfluxDB 2 server or cloud url, e.g. https://eu-central-1-1.aws.cloud2.influxdata.com (Use: InfluxDB UI -> Load Data -> Client Libraries)
#define INFLUXDB_URL "influxdb-url"
// InfluxDB 2 server or cloud API authentication token (Use: InfluxDB UI -> Load Data -> Tokens -> <select token>)
#define INFLUXDB_TOKEN "token"
// InfluxDB 2 organization name or id (Use: InfluxDB UI -> Settings -> Profile -> <name under tile> )
#define INFLUXDB_ORG "org"
// InfluxDB 2 bucket name (Use: InfluxDB UI -> Load Data -> Buckets)
#define INFLUXDB_BUCKET "bucket"
```

You need to pass an additional parameter to the client constructor, which is a certificate of the server to trust. The constant `InfluxDbCloud2CACert` contains the InfluxDB Cloud 2 CA certificate, which is predefined in this library:
```cpp
// Single InfluxDB instance
InfluxDBClient client(INFLUXDB_URL, INFLUXDB_ORG, INFLUXDB_BUCKET, INFLUXDB_TOKEN, InfluxDbCloud2CACert);
```
Read more about [secure connection](#secure-connection).

Additionally, time needs to be synced:
```cpp
// Synchronize time with NTP servers and set timezone
// Accurate time is necessary for certificate validation and writing in batches
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
InfluxDB client for Arduino can also write data in batches. A batch is simply a set of points that will be sent at once. To create a batch, the client will keep all points until the number of points reaches the batch size and then it will write all points at once to the InfluxDB server. This is often more efficient than writing each point separately.

### Timestamp
If using batch writes, the timestamp should be employed. Timestamp specifies the time when data was gathered and it is used in the form of a number of seconds (milliseconds, etc) from epoch (1.1.1970) UTC.
If points have no timestamp assigned, InfluxDB assigns a timestamp at the time of writing, which could happen much later than the data has been obtained, because the final batch write will happen when the batch is full (or when [flush buffer](#buffer-handling-and-retrying) is forced).

InfluxDB allows sending timestamps in various precisions - nanoseconds, microseconds, milliseconds or seconds. The milliseconds precision is usually enough for using on Arduino. The maximum available precision is microseconds. Setting the timestamp to nanoseconds will just add zeroes for microseconds fraction and will not improve timestamp accuracy.

The client has to be configured with a time precision. The default settings is to not use the timestamp, which means that the server will assign a timestamp when the data is written to the database. The `setWriteOptions` functions allows setting custom `WriteOptions` params and one of them is __write precision__:
``` cpp
// Set write precision to milliseconds. Leave other parameters default.
client.setWriteOptions(WriteOptions().writePrecision(WritePrecision::MS));
```
When a write precision is configured, the client will automatically assign the current time to the timestamp of each written point which doesn't have a timestamp assigned.

If you want to manage timestamp on your own, there are several ways to set the timestamp explicitly.
- `setTime(WritePrecision writePrecision)` - Sets the timestamp to the actual time in the desired precision
- `setTime(unsigned long long timestamp)` -  Sets the timestamp to an offset since the epoch. Correct precision must be set InfluxDBClient::setWriteOptions.
- `setTime(String timestamp)` - Sets the timestamp to an offset since the epoch. Correct precision must be set InfluxDBClient::setWriteOptions.

The `getTime()` method allows copying the timestamp between points.


### Configure Time
Dealing with timestamps, and also validating server or CA certificate, requires that the device has correctly set the time. This can be done with one line of code:
```cpp
// Synchronize time with NTP servers and set timezone
// Accurate time is necessary for certificate validation and writing in batches
// For the fastest time sync find NTP servers in your area: https://www.pool.ntp.org/zone/
configTzTime("PST8PDT", "pool.ntp.org", "time.nis.gov");
```
The `configTzTime` function starts the time synchronization with NTP servers. The first parameter specifies the timezone information, which is important for distinguishing between UTC and a local timezone and for daylight saving changes.
The last two string parameters are the internet addresses of NTP servers. Check [pool.ntp.org](https://www.pool.ntp.org/zone) for address of some local NTP servers.

Timezone string details are described at [https://www.gnu.org/software/libc/manual/html_node/TZ-Variable.html](https://www.gnu.org/software/libc/manual/html_node/TZ-Variable.html).
Values for some timezones:
- Central Europe: `CET-1CEST,M3.5.0,M10.5.0/3`
- Eastern: `EST5EDT`
- Japanese: `JST-9`
- Pacific Time: `PST8PDT`

There is also another function for syncing the time, which takes timezone and DST offset. As DST info is set via static offset it will create local time problem when DST change occurs.
It's declaration is following:
```cpp
configTime(long gmtOffset_sec, int daylightOffset_sec, const char* server1, const char* server2 = nullptr, const char* server3 = nullptr);
```

In the example code it would be:
```cpp
// Synchronize time with NTP servers
// Accurate time is necessary for certificate validation and writing in batches
configTime(3600, 3600, "pool.ntp.org", "time.nis.gov");
```

Both `configTzTime` and `configTime` functions are asynchronous. This means that calling the functions just starts the time synchronization. Time is often not synchronized yet upon returning from call.

There is a helper function `timeSync` provided with the this library. The function starts time synchronization by calling the `configTzTime` and waits maximum 20 seconds for time to be synchronized. It prints progress info and final local time to the `Serial` console.
`timeSync` has the same signature as `configTzTime` and it is included with the main header file `InfluxDbClient.h`:
```cpp
// Synchronize time with NTP servers and waits for competition. Prints waiting progress and final synchronized time to the Serial.
// Accurate time is necessary for certificate validation and writing points in batch
// For the fastest time sync find NTP servers in your area: https://www.pool.ntp.org/zone/
void timeSync(const char *tzInfo, const char* ntpServer1, const char* ntpServer2 = nullptr, const char* ntpServer3 = nullptr);
```

### Batch Size
Setting batch size depends on data gathering and DB updating strategy.

If data is written in short periods (seconds), the batch size should be set according to your expected write periods and update frequency requirements.
For example, if you would like to see updates (on the dashboard or in processing) each minute and you are measuring a single value (1 point) every 10s (6 points per minute), the batch size should be 6. If it is sufficient to update each hour and you are creating 1 point each minute, your batch size should be 60. The maximum recommended batch size is 200. Maximum batch size depends on the RAM of the device (80KB for ESP8266 and 512KB for ESP32).

In cases where the data should be written in longer periods and gathered data consists of several points, the batch size should be set to the expected number of points to be gathered.

To set the batch size we use `WriteOptions` object and  [setWriteOptions](#write-options) function:
```cpp
// Enable messages batching
client.setWriteOptions(WriteOptions().batchSize(10));
```
Writing the point will add a point to the underlying buffer until the batch size is reached:
```cpp
// Write first point to the buffer
// Buffered write always returns `true`
client.writePoint(point1);
// Write second point to the buffer
client.writePoint(point2);
..
// Write ninth point to the buffer
client.writePoint(point9);
// Writing tenth point will cause flushing buffer and returns actual write result.
if(!client.writePoint(point10)) {
    Serial.print("InfluxDB write failed: ");
    Serial.println(client.getLastErrorMessage());
}
```

In case cases where the number of points is not always the same, set the batch size to the maximum number of points and use the `flushBuffer()` function to force writing to the database. See [Buffer Handling](#buffer-handling-and-retrying) for more details.

## Buffer Handling and Retrying
InfluxDB contains an underlying buffer for handling writing in batches and automatic retrying on server back-pressure and connection failure.

Its size is controlled by the `bufferSize` param of [WriteOptions](#write-options) object:
```cpp
// Increase buffer to allow caching of failed writes
client.setWriteOptions(WriteOptions().bufferSize(50));
```
The recommended size is at least 2 x batch size.

The state of the buffer can be determined via two functions:
 - `isBufferEmpty()` - Returns true if buffer is empty
 - `isBufferFull()` - Returns true if buffer is full

 A full buffer can occur when there is a problem with the internet connection or the InfluxDB server is overloaded. In such cases, points to write remain in the buffer. When more points are added and connection problem remains, the buffer will reach the top and new points will overwrite older points.

 Each attempt to write a point will try to send older points in the buffer. So, the `isBufferFull()` function can be used to skip low priority points.

The `flushBuffer()` function can be used to force writing, even if the number of points in the buffer is lower than the batch size. With the help of the `isBufferEmpty()` function a check can be made before a device goes to sleep:

 ```cpp
  // Check whether buffer in not empty
  if (!client.isBufferEmpty()) {
      // Write all remaining points to db
      client.flushBuffer();
  }
```

Other functions for dealing with buffer:
 - `checkBuffer()` - Checks point buffer status and flushes if the number of points reaches batch size or flush interval runs out. This is the main function for controlling the buffer and it is used internally.
 - `resetBuffer()` - Clears the buffer.

Check [SecureBatchWrite example](examples/SecureBatchWrite/SecureBatchWrite.ino) for example code of buffer handling functions.

## Write Options
Writing points can be controlled via `WriteOptions`, which is set in the `setWriteOptions` function:

| Parameter | Default Value | Meaning |
|-----------|---------------|---------|
| writePrecision | `WritePrecision::NoTime` | Timestamp precision of written data |
| batchSize | `1` | Number of points that will be written to the database at once |
| bufferSize | `5` | Maximum number of points in buffer. Buffer contains new data that will be written to the database and also data that failed to be written due to network failure or server overloading |
| flushInterval | `60` | Maximum time(in seconds) data will be held in buffer before points are written to the db |

## HTTP Options
`HTTPOptions` controls some aspects of HTTP communication and they are set via `setHTTPOptions` function:
| Parameter | Default Value | Meaning |
|-----------|---------------|---------|
| reuseConnection | `false` | Whether HTTP connection should be kept open after initial communication. Usable for frequent writes/queries. |
| httpReadTimeout | `5000` | Timeout (ms) for reading server response |

## Secure Connection
Connecting to a secured server requires configuring the client to trust the server. This is achieved by providing the client with a server certificate, certificate authority certificate or certificate SHA1 fingerprint.

:memo: In ESP32 arduino SDK (1.0.4), `WiFiClientSecure` doesn't support fingerprint to validate the server certificate.

The certificate (in PEM format) or SHA1 fingerprint should be placed in flash memory to save RAM.
Code bellow is an example certificate in PEM format. Valid InfluxDB 2 Cloud CA certificate is included in the library in the constant `InfluxDbCloud2CACert`, located in the `InfluxDBCloud.h`.

You can use a custom server certificate by exporting it, e.g. using a web browser:
```cpp
// Server certificate in PEM format, placed in the program (flash) memory to save RAM
const char ServerCert[] PROGMEM =  R"EOF(
-----BEGIN CERTIFICATE-----
MIIFazCCA1OgAwIBAgIRAIIQz7DSQONZRGPgu2OCiwAwDQYJKoZIhvcNAQELBQAw
TzELMAkGA1UEBhMCVVMxKTAnBgNVBAoTIEludGVybmV0IFNlY3VyaXR5IFJlc2Vh
cmNoIEdyb3VwMRUwEwYDVQQDEwxJU1JHIFJvb3QgWDEwHhcNMTUwNjA0MTEwNDM4
WhcNMzUwNjA0MTEwNDM4WjBPMQswCQYDVQQGEwJVUzEpMCcGA1UEChMgSW50ZXJu
ZXQgU2VjdXJpdHkgUmVzZWFyY2ggR3JvdXAxFTATBgNVBAMTDElTUkcgUm9vdCBY
MTCCAiIwDQYJKoZIhvcNAQEBBQADggIPADCCAgoCggIBAK3oJHP0FDfzm54rVygc
h77ct984kIxuPOZXoHj3dcKi/vVqbvYATyjb3miGbESTtrFj/RQSa78f0uoxmyF+
0TM8ukj13Xnfs7j/EvEhmkvBioZxaUpmZmyPfjxwv60pIgbz5MDmgK7iS4+3mX6U
A5/TR5d8mUgjU+g4rk8Kb4Mu0UlXjIB0ttov0DiNewNwIRt18jA8+o+u3dpjq+sW
T8KOEUt+zwvo/7V3LvSye0rgTBIlDHCNAymg4VMk7BPZ7hm/ELNKjD+Jo2FR3qyH
B5T0Y3HsLuJvW5iB4YlcNHlsdu87kGJ55tukmi8mxdAQ4Q7e2RCOFvu396j3x+UC
B5iPNgiV5+I3lg02dZ77DnKxHZu8A/lJBdiB3QW0KtZB6awBdpUKD9jf1b0SHzUv
KBds0pjBqAlkd25HN7rOrFleaJ1/ctaJxQZBKT5ZPt0m9STJEadao0xAH0ahmbWn
OlFuhjuefXKnEgV4We0+UXgVCwOPjdAvBbI+e0ocS3MFEvzG6uBQE3xDk3SzynTn
jh8BCNAw1FtxNrQHusEwMFxIt4I7mKZ9YIqioymCzLq9gwQbooMDQaHWBfEbwrbw
qHyGO0aoSCqI3Haadr8faqU9GY/rOPNk3sgrDQoo//fb4hVC1CLQJ13hef4Y53CI
rU7m2Ys6xt0nUW7/vGT1M0NPAgMBAAGjQjBAMA4GA1UdDwEB/wQEAwIBBjAPBgNV
HRMBAf8EBTADAQH/MB0GA1UdDgQWBBR5tFnme7bl5AFzgAiIyBpY9umbbjANBgkq
hkiG9w0BAQsFAAOCAgEAVR9YqbyyqFDQDLHYGmkgJykIrGF1XIpu+ILlaS/V9lZL
ubhzEFnTIZd+50xx+7LSYK05qAvqFyFWhfFQDlnrzuBZ6brJFe+GnY+EgPbk6ZGQ
3BebYhtF8GaV0nxvwuo77x/Py9auJ/GpsMiu/X1+mvoiBOv/2X/qkSsisRcOj/KK
NFtY2PwByVS5uCbMiogziUwthDyC3+6WVwW6LLv3xLfHTjuCvjHIInNzktHCgKQ5
ORAzI4JMPJ+GslWYHb4phowim57iaztXOoJwTdwJx4nLCgdNbOhdjsnvzqvHu7Ur
TkXWStAmzOVyyghqpZXjFaH3pO3JLF+l+/+sKAIuvtd7u+Nxe5AW0wdeRlN8NwdC
jNPElpzVmbUq4JUagEiuTDkHzsxHpFKVK7q4+63SM1N95R1NbdWhscdCb+ZAJzVc
oyi3B43njTOQ5yOf+1CceWxG1bQVs5ZufpsMljq4Ui0/1lvh+wjChP4kqKOJ2qxq
4RgqsahDYVvTH9w7jXbyLeiNdd8XM2w9U/t7y0Ff/9yi0GE44Za4rF2LN9d11TPA
mRGunUHBcnWEvgJBQl9nJEiU0Zsnvgc/ubhPgXRR4Xq37Z0j4r7g1SgEEzwxA57d
emyPxgcYxn/eR44/KJ4EBs+lVDR3veyJm+kXQ99b21/+jh5Xos1AnX5iItreGCc=
-----END CERTIFICATE-----
)EOF";

// Alternatively, use a fingerprint of server certificate to set trust. Works only for ESP8266.
const char ServerCert[] PROGMEM = "cabd2a79a1076a31f21d253635cb039d4329a5e8";
```

### InfluxDb 2
There are two ways to set the certificate or fingerprint to trust a server:
 - Use full param constructor
```cpp
// InfluxDB client instance with preconfigured InfluxCloud certificate
InfluxDBClient client(INFLUXDB_URL, INFLUXDB_ORG, INFLUXDB_BUCKET, INFLUXDB_TOKEN, ServerCert);
```
- Use `setConnectionParams` function:
```cpp
// InfluxDB client instance
InfluxDBClient client;

void setup() {
    // configure client
    client.setConnectionParams(INFLUXDB_URL, INFLUXDB_ORG, INFLUXDB_BUCKET, INFLUXDB_TOKEN, ServerCert);
}
```
### InfluxDb 1

Use `setConnectionParamsV1` function:
```cpp
// InfluxDB client instance
InfluxDBClient client;

void setup() {
    // configure client
    client.setConnectionParamsV1(INFLUXDB_URL, INFLUXDB_DATABASE, INFLUXDB_USER, INFLUXDB_PASSWORD, ServerCert);
}
```
Another important prerequisite to successfully validate a server or CA certificate is to have properly synchronized time. More on this in [Configure Time](#configure-time).

:information_source: Time synchronization is not required for validating server certificate via SHA1 fingerprint.

### Skipping certificate validation
The CA certificate provided with the library is ISRG Root X1. This certificate lasts a very long time, until 2035. It is not necessary to update your device until then when using ISRG Root X1.

If you are using your own certificate, plase keep in mind server certificates have limited validity period, often only a few months. It will be necessary to frequently change trusted certificate in the source code and reflashing the device. A solution could be using OTA update, but you will still need to care about certificate validity and updating it ahead of time to avoid connection failures.

The best way to prevent frequent updates is to use a root certificate like the one provided with the library. If you are unable to use a root certificate from a trusted authority, you may want to use insecure mode instead. This is done with the help of `InfluxDBClient::setInsecure()` method. 
You will also save space in flash (and RAM) by leaving certificate param empty when calling constructor or `setConnectionParams` method.

:memo: The `InfluxDBClient::setInsecure()` method must be called before calling any function that will establish connection. The best place to call it is in the `setup` method: 

```cpp
// InfluxDB client instance without a server certificate
InfluxDBClient client(INFLUXDB_URL, INFLUXDB_ORG, INFLUXDB_BUCKET, INFLUXDB_TOKEN);

void setup() {
    // Set insecure connection to skip server certificate validation 
    client.setInsecure();
}
```

:warning: Using untrusted connection is a security risk.

## Querying
InfluxDB 2 and InfluxDB 1.7+ (with [enabled flux](https://docs.influxdata.com/influxdb/latest/administration/config/#flux-enabled-false)) uses [Flux](https://www.influxdata.com/products/flux/) to process and query data. InfluxDB client for Arduino offers a simple, but powerful, way how to query data with `query` function. It parses response line by line, so it can read a huge responses (thousands data lines), without consuming a lot device memory.

The `query` returns `FluxQueryResult` object, which parses response and provides useful getters for accessing values from result set.

The InfluxDB flux query result set is returned in CSV format. In the example below, the first line contains type information and the second column names, and the rest is data:
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

Accessing data using `FluxQueryResult` requires knowing the query result structure, especially the name and the type of the column. The best practice is to tune the query
in the `InfluxDB Data Explorer` and use the final query with this library.

 Browsing thought the result set is done by repeatedly calling the `next()` method, until it returns false. Unsuccessful reading is distinguished by a non empty value from the `getError()` method.
 As a flux query result can contain several tables, differing by grouping key, use the `hasTableChanged()` method to determine when there is a new table.
 Single values are returned using the `getValueByIndex()` or `getValueByName()` methods.
 All row values at once are retrieved by the `getValues()` method.
  Always call the `close()` method at the of reading.

A value in the flux query result column, retrieved by the `getValueByIndex()` or `getValueByName()` methods, is represented by the `FluxValue` object.
It provides getter methods for supported flux types:

| Flux type | Getter | C type |
| ----- | ------ |  --- |
| long | getLong() | long |
| unsignedLong | getUnsignedLong() | unsigned long |
| dateTime:RFC3339, dateTime:RFC3339Nano | getDateTime() |  [FluxDateTime](src/query/FluxTypes.h#L100) |
| bool | getBool() | bool |
| double | bool | double |
| string, base64binary, duration | getString() | String |

Calling improper type getter will result in a zero (empty) value.

Check for null (missing) value using the `isNull()` method.

Use the `getRawValue()` method for getting the original string form.

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

 Influxdb influx(INFLUXDB_HOST); // port defaults to 8086
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
influx.setPort(8086);
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
