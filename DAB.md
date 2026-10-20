# Device Automation Bus (DAB)

Version: 2.1  |  Last Updated: 2025-12-01

## 2.1 Release Notes

### New operations

- Added power mode operations:
  - `system/power-mode/get`: Get the current power mode of the device
  - `system/power-mode/set`: Set the device power mode (Active/Standby/Deep Sleep)
- Added application management operations:
  - `applications/install`: Install an application from a URL
  - `applications/uninstall`: Remove an application from the device
  - `applications/clear-data`: Clear all application data
  - `applications/install-from-app-store`: Install or update an application from the device's app store
- Added system operations:
  - `system/factory-reset`: Perform a full factory reset
  - `system/network-reset`: Reset all network settings
  - `system/logs/start-collection`: Begin collecting system logs
  - `system/logs/stop-collection`: Stop collecting logs and return them
- Added content operations:
  - `content/search`: Search for content available on the device
  - `content/recommendations`: Retrieve content recommendations
  - `content/open`: Open specified content
- Added new system settings:
  - `brightness`: Control screen brightness level
  - `contrast`: Control screen contrast level
  - `timeZone`: Set device time zone using IANA formats
  - `screenSaver`: Enable/disable screensaver
  - `screenSaverTimeout`: Set screensaver timeout in seconds
  - `screenSaverMinTimeout`: Minimum screensaver timeout value
  - `personalizedAds`: Enable/disable personalized advertisements
  - `highContrastText`: Enable/disable high contrast text
  - `identifierForAdvertising`: Control advertising identifier

### Enhancements

- Added requirements section (3.1-3.5) with miscellaneous compliance criteria
- Added operation latency requirements

### Documentation updates

- Renamed `Bluetooth` to `WirelessSpeakers` in AudioOutputSource enum to support a wider range of wireless audio devices
- Added `USB` as a video input source in VideoInputSource enum
- Removed deprecated APIs section to avoid confusion
- Renamed "language" setting to "system language" for clarity
- Added clarification that operations and functions are optional if the device doesn't support them

## Table of contents

1.  [Introduction](#1-introduction)
2.  [About the Specification](#2-about-the-specification)
3.  [Requirements](#3-requirements)
4.  [Protocol Basics](#4-protocol-basics)
5.  [DAB Operations](#5-dab-operations) \
    5.1. [Supported operations](#51-supported-operations) \
    5.2. [Applications](#52-applications) \
    5.3. [System](#53-system) \
    5.4. [Input](#54-input) \
    5.5. [Output](#55-output) \
    5.6.
    [Device and Application Telemetry](#56-device-and-application-telemetry) \
    5.7. [Health Check](#57-health-check) \
    5.8. [General Notifications](#58-optional-general-notifications) \
    5.9. [Voice](#59-voice) \
    5.10. [Content](#510-content)
6.  [Versioning of the protocol](#6-versioning-of-the-protocol)
7.  [Appendix](#7-appendix)

## 1. Introduction

This document describes Device Automation Bus (DAB), a simple protocol that lets
manufacturers and developers communicate with living room devices, such as
televisions and game consoles. These devices can connect to the Internet, start
and stop applications, send key presses, restart devices, and capture metrics.

With DAB, manufacturers and developers can automate tasks, such as compatibility
checks, performance testing, and monitoring memory. Automating tasks provides
reliable results, and saves time and money.

The following diagram provides a high-level view of how DAB works:

![DAB diagram](https://i.ibb.co/tLn05Gj/DAB-Diagram.png)

## 2. About the Specification

This specification describes a message-based protocol to support communicating
with a device under test. This specification describes the supported messages
and their payloads, and positive and negative response codes.

The DAB protocol is built on top of
[MQTT 5](https://docs.oasis-open.org/mqtt/mqtt/v5.0/mqtt-v5.0.html). MQTT is a
publish-subscribe messaging transport protocol for connecting to remote devices.
For more details, see https://mqtt.org/

### Not covered in the specification

The following topics are not covered in this specification:

#### Security

Recommendations for securing DAB and connecting to a device under test that
supports DAB will be defined in a separate document.

#### Protocol Compliance

Protocol Compliance testing will be defined in a separate DAB certification
guide.

## 3. Requirements

An endpoint is DAB compliant if it meets these conditions:

- 3.1 The endpoint MUST support
  [HTTP/1.1](https://datatracker.ietf.org/doc/html/rfc7230) or above for media
  operations
- 3.2 The DAB implementation MUST either be:
   - 3.2.1 Natively supported in your device firmware, or
   - 3.2.2 Supported using software that translates the DAB protocol into a
     protocol native to the device platform (aka Bridge implementation).
- 3.3 Native DAB implementations MUST use the default TCP/IP port 1883 to allow
  for uniform MQTT connection semantics across different test infrastructures.
- 3.4 Regardless of whether the implementation is Native or Bridge, operations
  MUST be completed within 200ms, unless otherwise noted in the DAB Operations
  documentation in section 5 of this spec.
- 3.5 DAB operations related to Application State management
  (e.g. app start, app stop, app state) MUST only provide the `200` response
  after the requested operation is completed.

## 4. Protocol Basics

This section describes message payloads, request and response formats, message
types, and common status codes.

## Request / Response Model

DAB relies on the
[MQTT request/response](https://docs.oasis-open.org/mqtt/mqtt/v5.0/os/mqtt-v5.0-os.html#_Toc3901252)
model, which is native to all MQTT 5 compliant brokers and clients.

To send requests and receive responses from DAB:

1.  The responding client (device side) is running and subscribed to all topics
    corresponding to the supported DAB operations.
2.  The requesting client (user side) determines to which topic responses should
    be sent and subscribes to this `responseTopic`. Response topics may be
    reused if `correlationData` is included, otherwise they must be unique.
3.  The requesting client generates a unique binary request ID, referred to as
    `correlationData`, to aid in associating responses with requests. Use of
    correlation data is not required, but strongly encouraged.
4.  The requesting client publishes the message to the request topic with the
    required `responseTopic` and optional `correlationData` message properties.
5.  When the request is received by the responding client, an operation is
    performed, response message is formed, `correlationData` is copied from the
    request message property to the response (if it exists), and the message is
    published to the `responseTopic` property from the request.
6.  The operation is complete when the requesting client receives a message on
    the `responseTopic` with matching `correlationData` (if applicable), or the
    client chooses to timeout the request.

### Example

1.  The user client wants to send a message to the request topic
    `dab/[DEVICE_ID]/health-check/get` and get responses at
    `dab/userClient-1/responses`.

    The `responseTopic` doesn't have to be similar to
    `dab/userClient-1/responses`, it can be anything that the user client
    specifies. Device client uses the `responseTopic` field to send back the
    response.

2.  A unique binary `correlationData` value is generated.

3.  The user client subscribes to `dab/userClient-1/responses` and waits for a
    response message with the matching `correlationData` value to arrive.

4.  The user client publishes a message to `dab/[DEVICE_ID]/health-check/get`
    with request properties `correlationData` and `responseTopic:
    dab/userClient-1/responses`.

5.  When a message sent by device client is received on the user client at
    `dab/userClient-1/responses` with matching `correlationData`, the message
    payload will be the health-check response and the operation is complete.

### Sample MQTT Request Message:

```json
{
   "topic": "dab/[DEVICE_ID]/health-check/get",
   "correlationData": "abc",
   "responseTopic": "dab/userClient-1/responses",
   "payload": "DabRequest{...}"
}
```

### Sample MQTT Response Message:

```json
{
   "topic": "dab/userClient-1/responses",
   "correlationData": "abc",
   "payload": "DabResponse{...}"
}
```

### Device ID Implementation Guidelines

The Device ID is a unique identifier that helps with DAB message routing across
multiple device clients.

Implement the Device ID as follows:

1.  The device under test defines its Device ID.
    *   1.1. It may or may not be baked into the device, but it should be
        globally unique.
    *   1.2. It may be configurable, but using transient values, such as current
        IP address, is not recommended as this can make reconnecting after
        restart difficult.
2.  The Device ID must be a string. It must contain only lowercase alphanumeric
    characters, hyphens (`-`), and underscores (`_`).

### Device Setup

When a device client connects to the MQTT broker, it should follow this setup
procedure:

1.  Subscribe to all topics listed in this spec, like
    `dab/[DEVICE_ID]/device/info` topic.
2.  Subscribe to `dab/discovery` topic.

### Device Discovery

A user client can discover all the DAB device clients present in the current
setup using these steps:

1.  A user client may connect to the MQTT broker when device clients are already
    connected. The user client should then publish a request to the discovery
    topic `dab/discovery`, similar to this:

    ```json
    {
       "topic": "dab/discovery",
       "correlationData": "abc",
       "responseTopic": "dab/userClient-1/responses",
       "payload": "DabRequest{...}"
    }
    ```

2.  All connected DAB devices will send a response back to the `responseTopic`
    set by the user client. The user client can then extract the Device ID from
    these responses.

### Message Payloads

DAB protocol message payloads are in JSON, using UTF-8 encoding.

This specification uses
[typescript interfaces](https://www.typescriptlang.org/docs/handbook/interfaces.html)
to define request and response schemas. TypeScript interfaces are used to be
descriptive, compact, and concise with the request and response schema
definitions.

As an example, the following JSON maps to the example TypeScript interface (and
vice versa):

```json
{
   "variableName": "Value",
}
```

maps to:

```typescript
interface ExampleResponse {
   variableName: string;
   optionallyIncludedVariable?: string;
}
```

For more information, see the
[TypeScript documentation](https://www.typescriptlang.org/docs).

### Common DAB Request Payload

All DAB requests must be valid JSON objects, even if empty. All DAB requests
must inherit from the `DabRequest` format

#### Request format

```typescript
interface DabRequest {
}
```

#### Sample response

An empty request is equal to an empty JSON map:

```json
{}
```

### Common DAB Response Payload

All DAB responses must inherit from the `DabResponse` format. They must contain
the `status` field, and may contain the `error` field if an error occurs.

#### Response format

```typescript
interface DabResponse {
   status: number;
   error?: string;
}
```

#### Response parameters

<table>
  <thead>
    <tr>
      <th>Parameter</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>status</td>
      <td>status code</td>
    </tr>
    <tr>
      <td>error</td>
      <td>optional explanation of the error</td>
    </tr>
  </tbody>
</table>

#### Common status codes

<table>
  <thead>
    <tr>
      <th>Status code</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>200</td>
      <td>The request was successful.</td>
    </tr>
    <tr>
      <td>400</td>
      <td>Bad request. The request was invalid or malformed. The explanation of the error must be included in the <code>error</code> field of the response.</td>
    </tr>
    <tr>
      <td>500</td>
      <td>Internal error. The explanation of the error must be included in the <code>error</code> field of the response.</td>
    </tr>
    <tr>
      <td>501</td>
      <td>Not implemented. The device must return this error when the requested functionality is not implemented.</td>
    </tr>
  </tbody>
</table>

Operations must respond with one of the common status codes. An operation may
use another code if specified.

#### Sample responses

A response with the success status code

```json
{
   "status": 200
}
```

A response with the error status code

```json
{
   "status": 400,
   "error": "App parameter is required to launch the application"
}
```

### Unsolicited Messages

Unsolicited messages are a feature of MQTT. An unsolicited message communicates
temporal information, like memory usage. The client subscribes to a topic that
receives these messages. The client may need to send a request to enable them.
Subscribing to the topic does not provide message history - the client will
receive only future messages.

### Data Types Definitions

<table>
  <thead>
    <tr>
      <th>Friendly name</th>
      <th>Type identifier</th>
      <th>Description</th>
      <th>Example</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Language code</td>
      <td><code>rfc_5646_language_tag</code></td>
      <td>All language tags use the IETF language tags defined in <a href="https://tools.ietf.org/html/rfc5646">RFC 5646</a>. All language tags are string fields.</td>
      <td>"en-US"</td>
    </tr>
    <tr>
      <td>UNIX Time stamp</td>
      <td><code>unix_timestamp_ms</code></td>
      <td>UNIX Epoch time in milliseconds. The data type is a 64 bit unsigned integer.</td>
      <td>1568815387403</td>
    </tr>
    <tr>
      <td>Application Identifier</td>
      <td><code>appId</code></td>
      <td>The application identifier consists of characters. For resiliency, uppercase letters are equivalent to lowercase (e.g., "netflix" is the same as "Netflix"). The minimum number of characters is 1 and the maximum is 64.</td>
      <td>"PrimeVideo"</td>
    </tr>
    <tr>
      <td>Uniform Resource Locator</td>
      <td><code>url</code></td>
      <td>MQTT has a max play load size of 256mb for binary data. HTTP is used for pushing and pulling resources from devices to support larger media files. URLs are defined in <a href="https://datatracker.ietf.org/doc/html/rfc3986">RFC 3986</a>.</td>
      <td>http://192.168.0.1/file</td>
    </tr>
    <tr>
      <td>Time Zone</td>
      <td><code>TimeZone</code></td>
      <td>All time zones are strings using the IANA formats defined in <a href="https://www.iana.org/time-zones">tzdb</a></td>
      <td>"Europe/Helsinki"</td>
    </tr>
  </tbody>
</table>

#### Standardized Application and Voice System Identifiers in Appendix

To standardize how applications and voice systems are referenced, key
identifiers and names are in [Section 7 -- Appendix](#7-appendix). DAB
implementations must use these App IDs and voice system names instead of
internal code names. This ensures consistent names in test automation across
DAB-capable platforms.

## 5. DAB Operations

### 5.1 Supported Operations

*Operation model: Request / Response*

#### Operation: List all supported operations

This operation lists all DAB operations that the device supports. All devices
must implement this operation. The operations are identified by the name of the
topic without the `dab` prefix. The result should not include the `list`
operations operation itself.

###### Request topic

`dab/[DEVICE_ID]/operations/list`

###### Request format

```typescript
type ListSupportedOperationRequest = DabRequest
```

###### Response format

```typescript
interface ListSupportedOperationResponse extends DabResponse {
   operations: string [];
}
```

###### Response parameters

<table>
  <thead>
    <tr>
      <th>Parameter</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>operations</td>
      <td>A list of operations supported by the DAB device client.</td>
    </tr>
  </tbody>
</table>

#### Sample response

```json
{
   "status": 200,
   "operations": [
      "applications/launch",
      "applications/launch-with-content",
      "applications/get-state",
      "applications/exit",
      "applications/install",
      "applications/uninstall",
      "applications/clear-data",
      "applications/install-from-app-store",
      "device/info",
      "system/power-mode/get",
      "system/power-mode/set",
      "system/restart",
      "system/factory-reset",
      "system/network-reset",
      "system/settings/list",
      "system/settings/get",
      "system/settings/set",
      "system/logs/start-collection",
      "system/logs/stop-collection",
      "input/key/list",
      "input/key-press",
      "input/long-key-press",
      "output/image",
      "device-telemetry/start",
      "device-telemetry/stop",
      "device-telemetry/metrics",
      "app-telemetry/start",
      "app-telemetry/stop",
      "app-telemetry/metrics",
      "health-check/get",
      "messages",
      "voice/list",
      "voice/send-audio",
      "voice/send-text",
   ]
}
```

### 5.2 Applications

#### Application states

An application can be in one of the following states:

<table>
  <thead>
    <tr>
      <th>State</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>STOPPED</td>
      <td>There is no instance of the application running on the device.</td>
    </tr>
    <tr>
      <td>FOREGROUND</td>
      <td>The application is visible, active and accepting input.</td>
    </tr>
    <tr>
      <td>BACKGROUND</td>
      <td>There is a running instance of the application on the device but it is not visible and not accepting input.</td>
    </tr>
  </tbody>
</table>

If your device supports other states, choose the closest match.

#### Operation: List all applications

*Operation model: Request / Response*

This operation lists installed applications. The list can be limited to
DAB-enabled applications.

###### Request topic

`dab/[DEVICE_ID]/applications/list`

###### Operation Latency

The operation MUST be completed within 250 milliseconds.

###### Request format

```typescript
type ListApplicationsRequest = DabRequest
```

###### Response format

```typescript
interface Application {
   appId: string;
   friendlyName?: string;
   version?: string;
}
```

<table>
  <thead>
    <tr>
      <th>Parameter</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>appId</td>
      <td>Application Identifier, as <a href="#7-appendix">defined in the DAB Registry</a></td>
    </tr>
    <tr>
      <td>friendlyName</td>
      <td><em>optional</em> application friendly name</td>
    </tr>
    <tr>
      <td>version</td>
      <td><em>optional</em> application version</td>
    </tr>
  </tbody>
</table>

```typescript
interface ListApplicationsResponse extends DabResponse {
   applications: Application [];
}
```

###### Response parameters

<table>
  <thead>
    <tr>
      <th>Parameter</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>applications</td>
      <td>A list of applications.</td>
    </tr>
  </tbody>
</table>

###### Sample response

```json
{
   "status": 200,
   "applications": [
      {
         "appId": "Netflix",
         "friendlyName": "Netflix",
         "version": "2.1.0"
      },
      {
         "appId": "PrimeVideo",
         "friendlyName": "Amazon Prime Video",
         "version": "2.1.0"
      },
      {
         "appId": "YouTube",
         "friendlyName": "YouTube",
         "version": "2.1.0"
      }
   ]
}
```

#### Operation: Launch an application

*Operation model: Request / Response*

This operation launches an instance of the application into the FOREGROUND
state. There can only be one instance of an application running in the system
and the instance is referenced by the application identifier. Send the response
only after the application state changes on the platform.

*   If the application is in the STOPPED state, this operation launches a new
    instance into the FOREGROUND state.
*   If the application is in the BACKGROUND state, this operation brings the
    instance of the application into the FOREGROUND state.
*   If the application is in the FOREGROUND state, this operation doesn't change
    the state of the application.


###### Request topic

`dab/[DEVICE_ID]/applications/launch`

###### Request format

```typescript
interface LaunchApplicationRequest extends DabRequest {
   appId: string;
   parameters?: string [];
}
```

###### Request Parameters

<table>
  <thead>
    <tr>
      <th>Parameter</th>
      <th>Default</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>appId</td>
      <td>-</td>
      <td>Application Identifier</td>
    </tr>
    <tr>
      <td>parameters</td>
      <td><code>[]</code></td>
      <td>URL encoded list of parameters to be passed to the application being launched (usually directly to the app executable). The technique to pass parameters may be app specific. Work with <a href="#7-appendix">app partners in DAB Registry</a> for how the parameters are expected to be passed.</td>
    </tr>
  </tbody>
</table>

###### Sample request

```json
{
   "appId": "Netflix",
   "parameters": [
    "-KEY",
    "https%3A%2F%2Fwww.example-value.com%2F",
    "-STANDALONE_PARAM",
    "%22param-with-quotes%22",
   ]
}
```

In this sample, since `parameters` is defined, DAB implementations must URL
decode each parameter and pass them in order. For example, Netflix would
receive: `./netflix -KEY https://www.example-value.com -STANDALONE_PARAM
"param-with-quotes"`. URL encoding preserves parameter order and special
characters.

###### Response format

```typescript
type LaunchApplicationResponse = DabResponse
```

#### Operation: Launch an application to the specified content

*Operation model: Request / Response*

This operation launches the application into FOREGROUND state with the specified
content (also known as deep linking). The application identifier references a
single running instance. Send the response after the application state changes.

*   If the application is in the STOPPED state, this operation will launch a new
    instance into the FOREGROUND state with the specified content
*   If the application is in the BACKGROUND state, this operation will bring the
    instance of the application into the FOREGROUND and pass the specified
    content to the instance
*   If the application is in the FOREGROUND state, this operation will pass the
    content to the instance

###### Request topic

`dab/[DEVICE_ID]/applications/launch-with-content`

###### Request format

```typescript
interface LaunchApplicationWithContentRequest extends DabRequest {
   appId: string;
   contentId: string;
   parameters?: string [];
}
```

###### Request Parameters

<table>
  <thead>
    <tr>
      <th>Parameter</th>
      <th>Default</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>appId</td>
      <td>-</td>
      <td>Application Identifier</td>
    </tr>
    <tr>
      <td>contentId</td>
      <td>-</td>
      <td>Application specific content identifier</td>
    </tr>
    <tr>
      <td>parameters</td>
      <td><code>[]</code></td>
      <td>URL encoded list of parameters to be passed to the application being launched (usually directly to the app executable). The technique to pass parameters may be app specific. Work with <a href="#7-appendix">app partners in DAB Registry</a> for how the parameters are expected to be passed.</td>
    </tr>
  </tbody>
</table>

###### Sample request

```json
{
   "appId": "YouTube",
   "contentId": "XqZsoesa55w"
}
```

###### Response format

```typescript
type LaunchApplicationWithContentResponse = DabResponse
```

#### Operation: Get the application state

*Operation model: Request / Response*

This operation returns the state the application is in.

###### Request topic

`dab/[DEVICE_ID]/applications/get-state`

###### Request format

```typescript
interface GetApplicationStateRequest extends DabRequest {
   appId: string;
}
```

###### Request Parameters

<table>
  <thead>
    <tr>
      <th>Parameter</th>
      <th>Default</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>appId</td>
      <td>-</td>
      <td>Application Identifier</td>
    </tr>
  </tbody>
</table>

###### Sample request

```json
{
   "appId": "PrimeVideo"
}
```

###### Response format

```typescript
interface GetApplicationStateResponse extends DabResponse {
   state: string;
}
```

###### Response parameters

<table>
  <thead>
    <tr>
      <th>Parameter</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>state</td>
      <td>State of the application at the time of the request, one of <code>STOPPED</code>, <code>BACKGROUND</code> or <code>FOREGROUND</code></td>
    </tr>
  </tbody>
</table>

###### Sample response

```json
{
   "status": 200,
   "state": "BACKGROUND"
}
```

#### Operation: Exit the application

*Operation model: Request / Response*

This operation exits the application.

*   If the optional *background* parameter is set, this operation attempts to
    move the application to the BACKGROUND state.
*   If the *background* parameter is omitted or is set to false, this operation
    attempts to completely stop the application and move it to a STOPPED state.

When using the exit operation without the background flag, the intention is to
stop/kill the corresponding application process. When using the exit operation
with the background flag, the intention is to put the application in a
background state, similarly to when the user presses the `home` key. The
response should only be sent after the application state has changed on the
platform.

###### Request topic

`dab/[DEVICE_ID]/applications/exit`

###### Request format

```typescript
interface ExitApplicationRequest extends DabRequest {
   appId: string;
   background?: boolean;
}
```

###### Request Parameters

<table>
  <thead>
    <tr>
      <th>Parameter</th>
      <th>Default</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>appId</td>
      <td>-</td>
      <td>Application Identifier</td>
    </tr>
    <tr>
      <td>background</td>
      <td><code>false</code></td>
      <td>Decides whether to background or completely stop the application</td>
    </tr>
  </tbody>
</table>

###### Sample request

```json
{
   "appId": "Netflix",
   "background": false
}
```

###### Response format

```typescript
interface ExitApplicationResponse extends DabResponse {
   state: string;
}
```

###### Response parameters

<table>
  <thead>
    <tr>
      <th>Parameter</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>state</td>
      <td>State of the application after the exit operation</td>
    </tr>
  </tbody>
</table>

###### Sample response

```json
{
   "status": 200,
   "state": "STOPPED"
}
```

#### Operation: Install the application

*Operation model: Request / Response*

This operation sideloads the application from the request URL. If already
installed, it will uninstall and then reinstall the application.

The URL must be an HTTP or HTTPS URL. Local file paths are not allowed.

###### Request topic

`dab/[DEVICE_ID]/applications/install`

###### Operation Latency

The operation MUST be completed within 60 seconds.

###### Request format

```typescript
interface InstallApplicationRequest extends DabRequest {
   appId: string;
   url: string;
   format?: string;
   timeout?: number;
}
```

###### Request Parameters

<table>
  <thead>
    <tr>
      <th>Parameter</th>
      <th>Default</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>appId</td>
      <td>-</td>
      <td>Application Identifier</td>
    </tr>
    <tr>
      <td>url</td>
      <td>-</td>
      <td>Application package HTTP or HTTPS URL. Local file paths are not allowed.</td>
    </tr>
    <tr>
      <td>format</td>
      <td>-</td>
      <td><em>optional</em> Application package format. This only applies if there are multiple different formats for an installation.</td>
    </tr>
    <tr>
      <td>timeout</td>
      <td>-</td>
      <td><em>optional</em> Action timeout parameter in seconds. If the installation takes longer than the timeout, your implementation must stop the installation process and return a 408 timeout response.</td>
    </tr>
  </tbody>
</table>

###### Sample request

```typescript
{
   "appId": "YouTube",
   "url": "http://192.168.2.108/app/youtube",
   "format": "apk"
}
```

###### Response format

```typescript
type InstallApplicationResponse = DabResponse
```

#### Operation: Uninstall the application

*Operation model: Request / Response*

This operation uninstalls the application. This operation should also delete
local storage. If uninstallation fails, the device should response with a 403
error.

###### Request topic

`dab/[DEVICE_ID]/applications/uninstall`

###### Operation Latency

The operation MUST be completed within 10 seconds.

###### Request Format

```typescript
interface UninstallApplicationRequest extends DabRequest {
   appId: string;
}
```

###### Request Parameters

<table>
  <thead>
    <tr>
      <th>Parameter</th>
      <th>Default</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>appId</td>
      <td>-</td>
      <td>Application Identifier</td>
    </tr>
  </tbody>
</table>

###### Sample request

```typescript
{
   "appId": "YouTube"
}
```

###### Response format

```typescript
type UninstallApplicationResponse = DabResponse
```

#### Operation: Clear the application data

*Operation model: Request / Response*

This operation clears the application data.

###### Request topic

`dab/[DEVICE_ID]/applications/clear-data`

###### Request format

```typescript
interface ClearApplicationDataRequest extends DabRequest {
   appId: string;
}
```

###### Request Parameters

<table>
  <thead>
    <tr>
      <th>Parameter</th>
      <th>Default</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>appId</td>
      <td>-</td>
      <td>Application Identifier</td>
    </tr>
  </tbody>
</table>

###### Sample request

```typescript
{
   "appId": "YouTube"
}
```

###### Response format

```typescript
type ClearApplicationDataResponse = DabResponse
```

#### Operation: Install the application from App Store

*Operation model: Request / Response*

This operation installs the application from the app store. It updates the app
if the app is already installed. A 401 error indicates required user login.

###### Request topic

`dab/[DEVICE_ID]/applications/install-from-app-store`

###### Operation Latency

The operation MUST be completed within 60 seconds.

###### Request format

```typescript
interface InstallFromAppStoreRequest extends DabRequest {
   appId: string;
   appStoreId?: string;
}
```

###### Request Parameters

<table>
  <thead>
    <tr>
      <th>Parameter</th>
      <th>Default</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>appId</td>
      <td>-</td>
      <td>The unique identifier of the application.</td>
    </tr>
    <tr>
      <td>appStoreId</td>
      <td>-</td>
      <td><em>optional</em> The app store URL or unique store identifier for locating and installing the application.</td>
    </tr>
  </tbody>
</table>

###### Sample request

```typescript
{
   "appId": "YouTube",
   "appStoreId": "https://play.google.com/store/apps/details?id=com.google.android.youtube"
}
```

###### Response format

```typescript
type InstallFromAppStoreResponse = DabResponse
```

### 5.3. System

#### Operation: Device information

*Operation model: Request / Response*

###### Request topic

`dab/[DEVICE_ID]/device/info`

###### Request format

```typescript
type GetDeviceInformationRequest = DabRequest
```

###### Response format

```typescript
enum NetworkInterfaceType {
   Ethernet = "Ethernet",
   Wifi = "Wifi",
   Bluetooth = "Bluetooth",
   Coax = "Coax",
   Other = "Other",
}

enum DisplayType {
   // Device has native display capability
   Native = "Native",
   // Device cannot display natively and needs an external display connected
   External = "External",
}

interface NetworkInterface {
   connected: boolean;
   macAddress: string;
   ipAddress?: string;
   dns?: string [];
   type: NetworkInterfaceType;
}

interface GetDeviceInformationResponse extends DabResponse {
   manufacturer: string;
   model: string;
   serialNumber: string;
   chipset: string;
   firmwareVersion: string;
   firmwareBuild: string;
   networkInterfaces: NetworkInterface [];
   displayType: DisplayType;
   screenWidthPixels: number;
   screenHeightPixels: number;
   uptimeSince: unix_timestamp_ms;
   deviceId: string;
}
```

###### Response parameters

<table>
  <thead>
    <tr>
      <th>Parameter</th>
      <th>Description</th>
      <th>Example</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>manufacturer</td>
      <td>The device manufacturer</td>
      <td>Samsung</td>
    </tr>
    <tr>
      <td>model</td>
      <td>The device model</td>
      <td>UE32F4500</td>
    </tr>
    <tr>
      <td>serialNumber</td>
      <td>String used by the manufacturer to uniquely identify this device</td>
      <td>abc123</td>
    </tr>
    <tr>
      <td>chipset</td>
      <td>Device chipset</td>
      <td>X12</td>
    </tr>
    <tr>
      <td>firmwareVersion</td>
      <td>Device OS or firmware version</td>
      <td>7.00.</td>
    </tr>
    <tr>
      <td>firmwareBuild</td>
      <td>Device OS or firmware build number</td>
      <td>60129</td>
    </tr>
    <tr>
      <td>networkInterfaces</td>
      <td>One or more network interfaces, indicating the type, connection status, MAC address, and IP address (if applicable)</td>
      <td>WiFi, Ethernet, Bluetooth</td>
    </tr>
    <tr>
      <td>displayType</td>
      <td>Type of display used by the device.</td>
      <td>TVs, monitors, projectors; HDMI/DisplayPort displays for STBs, streaming sticks, soundbars, etc.</td>
    </tr>
    <tr>
      <td>screenWidthPixels</td>
      <td>Current screen resolution width measured in pixels. Should not be present when no screen is attached.</td>
      <td>1920</td>
    </tr>
    <tr>
      <td>screenHeightPixels</td>
      <td>Current screen resolution height measured in pixels. Should not be present when no screen is attached.</td>
      <td>1080</td>
    </tr>
    <tr>
      <td>uptimeSince</td>
      <td>The Unix timestamp of when the device last booted.</td>
      <td>1615205137769 (represents a boot time)</td>
    </tr>
    <tr>
      <td>deviceId</td>
      <td>The unique identifier of this device. Should be same as <code>[DEVICE_ID]</code> in the topic.</td>
      <td>device1</td>
    </tr>
  </tbody>
</table>

If any parameters are not applicable or unsupported, do not include them in the
response.

#### Operation: Get device power mode

*Operation model: Request / Response*

Gets the current power mode of the device.

<table>
  <thead>
    <tr>
      <th>Power Mode</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Active</td>
      <td>The device is fully operational.</td>
    </tr>
    <tr>
      <td>Standby</td>
      <td>The device is in a low-power state, ready to quickly resume operation.</td>
    </tr>
    <tr>
      <td>DeepSleep</td>
      <td>The device is in an ultra low-power state, with only critical systems remaining operational. 
        Devices supporting Wake-on-LAN technology should still capable of being awaken from this state.</td>
    </tr>
  </tbody>
</table>

###### Request topic

`dab/[DEVICE_ID]/system/power-mode/get`

###### Request format

```typescript
enum PowerMode {
   Active = "Active",
   Standby = "Standby",
   DeepSleep = "Deep Sleep",
}

type GetPowerModeRequest = DabRequest;
```

###### Response format

```typescript
interface GetPowerModeResponse extends DabResponse {
   powerMode: PowerMode;
}
```

###### Response parameters

<table>
  <thead>
    <tr>
      <th>Parameter</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>powerMode</td>
      <td>The current power mode of the device.</td>
    </tr>
  </tbody>
</table>

###### Sample response

```json
{
   "status": 200,
   "powerMode": "Active"
}
```

#### Operation: Set device power mode

*Operation model: Request / Response*

Sets the power mode of the device. The operation returns a response after the
device has been switched to the specified power mode. DAB must remain active
regardless of the power mode. If the device is already in the requested power
mode, its status is maintained, and a response is returned. Upon completion, the
operation returns the device's current power mode.

###### Request topic

`dab/[DEVICE_ID]/system/power-mode/set`

###### Request format

```typescript
interface SetPowerModeRequest extends DabRequest {
   powerMode: PowerMode;
}
```

###### Request Parameters

<table>
  <thead>
    <tr>
      <th>Parameter</th>
      <th>Default</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>powerMode</td>
      <td>-</td>
      <td>The power mode to which the device is set.</td>
    </tr>
  </tbody>
</table>

###### Sample request

```json
{
   "powerMode": "Standby"
}
```

###### Response format

```typescript
interface SetPowerModeResponse extends DabResponse {
   powerMode: PowerMode;
}
```

###### Response parameters

<table>
  <thead>
    <tr>
      <th>Parameter</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>powerMode</td>
      <td>The current power mode of the device.</td>
    </tr>
  </tbody>
</table>

###### Sample response

```json
{
   "status": 200,
   "powerMode": "Standby"
}
```

#### Operation: Restart the device

*Operation model: Request / Response*

This operation will start the device restart sequence.
After the device is restarted, DAB should stay enabled;
DAB remaining enabled is the only expectation of the specification.
The response should be sent after this operation is initiated.

###### Request topic

`dab/[DEVICE_ID]/system/restart`

###### Request format

```typescript
type RestartDeviceRequest = DabRequest
```

###### Response format

```typescript
type RestartDeviceResponse = DabResponse
```

Success:

```json
{
   "status": 200
}
```

Error:

```json
{
   "status": 500,
   "error": "insufficient privileges"
}
```

#### Operation: System Factory Reset

*Operation model: Request / Response*

This operation must invoke a full factory reset, similar to the UI menu option
that users can select. DAB response must be provided prior to commencing the
factory reset. DAB must remain enabled after factory reset. The DAB Device ID
must not change after the reset. It is recognized that loss of network
(i.e. WiFi) may require users to intervene after factory reset. The response
should be sent after this operation is initiated.

###### Request topic

`dab/[DEVICE_ID]/system/factory-reset`

###### Request Topic

`dab/<device-id>/system/factory-reset`

###### Operation Latency

The operation MUST be completed within 10 minutes.

###### Request format

```typescript
type FactoryResetDeviceRequest = DabRequest
```

###### Response Format

```typescript
type FactoryResetDeviceResponse = DabResponse
```

Success:
```json
{
   "status": 200
}
```

Error:
```json
{
   "status": 500,
   "error": "The device ran into an internal issue while performing the reset. Aborting."
}
```

#### Operation: System Network Reset

*Operation model: Request / Response*

This operation resets all network settings on the device. It must remove all
stored network configurations (such as Wi-Fi credentials, encryption keys, and
DHCP leases) and revert network settings to their defaults. This ensures the
device reestablishes network connections from scratch, re-resolving anything
tied to previous configurations. DAB must remain functional after the reset. The
response should be sent after this operation is initiated.

###### Request topic

`dab/[DEVICE_ID]/system/network-reset`

###### Request format

```typescript
type NetworkResetRequest = DabRequest
```

###### Response format

```typescript
type NetworkResetResponse = DabResponse
```

Sample success:

```json
{
    "status": 200
}
```

Sample error:

```json
{
    "status": 500,
    "error": "Failed to restart network module"
}
```

#### Operation: List system settings

*Operation model: Request / Response*

This operation is used to retrieve the possible values that can be assigned to
each system setting.

Operations and functions are optional if the device doesn't support them. If any parameters are not applicable or unsupported by the device, they should not be included in the response.

###### Request topic

`dab/[DEVICE_ID]/system/settings/list`

###### Request format

```typescript
type ListSupportedSystemSettingsRequest = DabRequest
```

###### Response format

```typescript
interface OutputResolution {
   width: number;
   height: number;
   frequency: number;
}

enum MatchContentFrameRate {
    // Match Content Frame Rate is Always Allowed even if the transition is not Seamless
    EnabledAlways = "EnabledAlways",
    // Match Content Frame Rate is Only Allowed when the transition is Seamless
    EnabledSeamlessOnly = "EnabledSeamlessOnly",
    // Match Content Frame Rate is Disabled
    Disabled = "Disabled",
}

enum HdrOutputMode {
    // Always output HDMI signal in HDR Format
    AlwaysHdr = "AlwaysHdr",
    // Output HDMI signal in HDR Format Only when playing HDR content
    HdrOnPlayback = "HdrOnPlayback",
    // Never Output HDMI Signal in HDR Mode
    DisableHdr = "DisableHdr",
}

enum PictureMode {
    // Standard Mode or Normal Mode
    Standard = "Standard",
    // Dynamic or Vivid Mode
    Dynamic = "Dynamic",
    // Movie or Cinema Mode
    Movie = "Movie",
    // Sports Mode
    Sports = "Sports",
    // Filmmaker Mode
    FilmMaker = "FilmMaker",
    // Game Mode, Lowest Latency Option
    Game = "Game",
    // Automatically adjust the picture mode to the best option for current content
    Auto = "Auto",
}

enum AudioOutputMode {
    // 2 Channel PCM
    Stereo = "Stereo",
    // Multichannel PCM e.g. 5.1 PCM
    MultichannelPcm = "MultichannelPcm",
    // Bitstreams sent in passthrough mode
    PassThrough = "PassThrough",
    // Audio Output Mode Recommended by the device partner in current setup
    Auto = "Auto",
}

enum AudioOutputSource {
    // Audio Output on Native Speakers of the device
    NativeSpeaker = "NativeSpeaker",
    // Audio Output on ARC Port
    Arc = "Arc",
    // Audio Output on E-ARC Port
    EArc = "EArc",
    // Audio Output on Optical Port
    Optical = "Optical",
    // Audio Output on Auxiliary Port
    Aux = "Aux",
    // Audio Output on connected Wireless Speakers
    WirelessSpeakers = "WirelessSpeakers",
    // Audio Output in the default mode recommended by the device partner in current setup
    Auto = "Auto",
    // HDMI, This is only applicable for Source devices
    HDMI = "HDMI",
}

enum VideoInputSource {
    // TV Tuner / Antenna
    Tuner = "Tuner",
    // First HDMI Input Port
    HDMI1 = "HDMI1",
    // Second HDMI Input Port
    HDMI2 = "HDMI2",
    // Third HDMI Input Port
    HDMI3 = "HDMI3",
    // Fourth HDMI Input Port
    HDMI4 = "HDMI4",
    // Composite input port
    Composite = "Composite",
    // Component input port
    Component = "Component",
    // USB input port
    USB = "USB",
    // TV Home Screen
    Home = "Home",
    // Wireless Casting
    Cast = "Cast",
}

interface ListSupportedSystemSettingsResponse extends DabResponse {
   language: rfc_5646_language_tag [];
   outputResolution: OutputResolution [];
   memc: boolean;
   cec: boolean;
   lowLatencyMode: boolean;
   matchContentFrameRate: MatchContentFrameRate [];
   hdrOutputMode: HdrOutputMode [];
   pictureMode: PictureMode [];
   brightness: {
      min: number;
      max: number;
   },
   contrast: {
      min: number;
      max: number;
   },
   audioOutputMode: AudioOutputMode [];
   audioOutputSource: AudioOutputSource [];
   videoInputSource: VideoInputSource [];
   audioVolume: {
      min: number;
      max: number;
   },
   mute: boolean;
   timeZone: TimeZone [];
   textToSpeech: boolean;
   screenSaver: boolean;
   screenSaverMinTimeout: number;
   personalizedAds: boolean;
   highContrastText: boolean;
   identifierForAdvertising: boolean;
}
```

###### Response parameters

<table>
  <thead>
    <tr>
      <th>Setting</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>language</td>
      <td>All possible system language settings per RFC 5646 language tag</td>
    </tr>
    <tr>
      <td>outputResolution</td>
      <td>All possible Video output resolutions for the device</td>
    </tr>
    <tr>
      <td>memc</td>
      <td>Motion estimation and compensation setting support</td>
    </tr>
    <tr>
      <td>cec</td>
      <td>CEC feature support on the device</td>
    </tr>
    <tr>
      <td>lowLatencyMode</td>
      <td>Low Latency Mode Setting support</td>
    </tr>
    <tr>
      <td>matchContentFrameRate</td>
      <td>Match Content FrameRate selection options</td>
    </tr>
    <tr>
      <td>hdrOutputMode</td>
      <td>HDR Output Mode options on the Source Device</td>
    </tr>
    <tr>
      <td>pictureMode</td>
      <td>Picture modes supported by the device</td>
    </tr>
    <tr>
      <td>audioOutputMode</td>
      <td>Audio Output modes supported by the device</td>
    </tr>
    <tr>
      <td>audioOutputSource</td>
      <td>Audio output source selection options</td>
    </tr>
    <tr>
      <td>videoInputSource</td>
      <td>Video input source options on TV</td>
    </tr>
    <tr>
      <td>audioVolume</td>
      <td>Audio output volume control. Indicates whether the feature is supported</td>
    </tr>
    <tr>
      <td>mute</td>
      <td>Volume mute operation support on the device</td>
    </tr>
    <tr>
      <td>timeZone</td>
      <td>All supported time zone values using the IANA formats</td>
    </tr>
    <tr>
      <td>textToSpeech</td>
      <td>Text to Speech support on the device</td>
    </tr>
    <tr>
      <td>screenSaver</td>
      <td>Screensaver support on the device</td>
    </tr>
    <tr>
      <td>screenSaverMinTimeout</td>
      <td>Minimum screensaver timeout in seconds (maximum 60).</td>
    </tr>
    <tr>
      <td>personalizedAds</td>
      <td>Personalized Ads support on the device</td>
    </tr>
    <tr>
      <td>highContrastText</td>
      <td>High Contrast Text Setting support</td>
    </tr>
    <tr>
      <td>identifierForAdvertising</td>
      <td>Personalized Ads identifier</td>
    </tr>
  </tbody>
</table>

For settings with boolean values:

-   `true`: The feature is supported.
-   `false`: The feature is not supported.

For settings with minimum/maximum values:

- `min`: Minimum supported value for the setting.
- `max`: Maximum supported value for the setting.

###### Sample response

```json
{
   "status": 200,
   "language": ["en-US", "es"],
   "outputResolution": [{
      "width": 3840,
      "height": 2160,
      "frequency": 60
   },
   {
      "width": 1920,
      "height": 1080,
      "frequency": 60
   }],
   "memc": false,
   "cec": true,
   "lowLatencyMode": true,
   "matchContentFrameRate": ["EnabledAlways", "EnabledSeamlessOnly","Disabled"],
   "hdrOutputMode": ["AlwaysHdr", "HdrOnPlayback", "DisableHdr"],
   "pictureMode": ["Standard", "Dynamic", "Movie"],
   "brightness": {
      "min": 1,
      "max": 100
   },
   "contrast": {
      "min": 0,
      "max": 50
   },
   "audioOutputMode": ["Stereo", "Auto"],
   "audioOutputSource": ["HDMI", "Optical"],
   "videoInputSource": [],
   "audioVolume": {
      "min": 0,
      "max": 100
   },
   "mute": false,
   "timeZone": ["America/New_York", "America/Los_Angeles", "Europe/London", "Asia/Tokyo"],
   "textToSpeech": true,
   "screenSaver": true,
   "screenSaverMinTimeout": 5,
   "personalizedAds": true,
   "highContrastText": true,
   "identifierForAdvertising": true
}
```

#### Operation: Get system settings

*Operation model: Request / Response*

This operation is used to retrieve the current settings on the device.

```typescript
interface SystemSettings {
   language: rfc_5646_language_tag;
   outputResolution: OutputResolution;
   memc: boolean;
   cec: boolean;
   lowLatencyMode: boolean;
   matchContentFrameRate: MatchContentFrameRate;
   hdrOutputMode: HdrOutputMode;
   pictureMode: PictureMode;
   brightness: number;
   contrast: number;
   audioOutputMode: AudioOutputMode;
   audioOutputSource: AudioOutputSource;
   videoInputSource: VideoInputSource;
   audioVolume: number;
   mute: boolean;
   timeZone: TimeZone;
   textToSpeech: boolean;
   screenSaver: boolean;
   screenSaverTimeout: number;
   personalizedAds: boolean;
   highContrastText: boolean;
   identifierForAdvertising: string;
}
```

<table>
  <thead>
    <tr>
      <th>Setting</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>language</td>
      <td>Current system language setting per RFC 5646 language tag</td>
    </tr>
    <tr>
      <td>outputResolution</td>
      <td>Current Video output resolution for the device</td>
    </tr>
    <tr>
      <td>memc</td>
      <td>Current Motion estimation and compensation setting used</td>
    </tr>
    <tr>
      <td>cec</td>
      <td>Current CEC feature state on the device</td>
    </tr>
    <tr>
      <td>lowLatencyMode</td>
      <td>Current Low Latency Mode Setting used on the device</td>
    </tr>
    <tr>
      <td>matchContentFrameRate</td>
      <td>Current Match Content FrameRate selected option</td>
    </tr>
    <tr>
      <td>hdrOutputMode</td>
      <td>HDR Output Mode as selected on the Source Device</td>
    </tr>
    <tr>
      <td>pictureMode</td>
      <td>Picture modes currently used by the device</td>
    </tr>
    <tr>
      <td>brightness</td>
      <td>Current brightness level</td>
    </tr>
    <tr>
      <td>contrast</td>
      <td>Current contrast level</td>
    </tr>
    <tr>
      <td>audioOutputMode</td>
      <td>Audio Output mode currently used by the device</td>
    </tr>
    <tr>
      <td>audioOutputSource</td>
      <td>Current Audio output source selected on the device</td>
    </tr>
    <tr>
      <td>videoInputSource</td>
      <td>Current Video input source selected on TV</td>
    </tr>
    <tr>
      <td>audioVolume</td>
      <td>Current audio output volume</td>
    </tr>
    <tr>
      <td>mute</td>
      <td>Current Volume mute state</td>
    </tr>
    <tr>
      <td>timeZone</td>
      <td>Current time zone</td>
    </tr>
    <tr>
      <td>textToSpeech</td>
      <td>Current Text to Speech state</td>
    </tr>
    <tr>
      <td>screenSaver</td>
      <td>Current screensaver setting state</td>
    </tr>
    <tr>
      <td>screenSaverTimeout</td>
      <td>Current screensaver timeout setting in number of seconds</td>
    </tr>
    <tr>
      <td>personalizedAds</td>
      <td>Current personalized Ads state (disabled = the user opted out)</td>
    </tr>
    <tr>
      <td>highContrastText</td>
      <td>Current High Contrast Text state</td>
    </tr>
    <tr>
      <td>identifierForAdvertising</td>
      <td>Current identifier for advertising string</td>
    </tr>
  </tbody>
</table>

For settings with boolean values:

- `true`: the feature is enabled
- `false`: the feature is disabled

##### Request topic

`dab/[DEVICE_ID]/system/settings/get` (dab: Device Abstraction Bus)

##### Request format

```typescript
type GetCurrentSystemSettingsRequest = DabRequest
```

##### Response format

```typescript
interface GetCurrentSystemSettingsResponse extends DabResponse, SystemSettings {}
```

##### Sample response

```json
{
   "status": 200,
   "language": "en-US",
   "outputResolution": {
      "width": 3840,
      "height": 2160,
      "frequency": 60
   },
   "memc": false,
   "cec": true,
   "lowLatencyMode": true,
   "matchContentFrameRate": "EnabledSeamlessOnly",
   "hdrOutputMode": "AlwaysHdr",
   "pictureMode": "Standard",
   "brightness": 55,
   "contrast": 50,
   "audioOutputMode": "Auto",
   "audioOutputSource": "HDMI",
   "videoInputSource": "HDMI1",
   "audioVolume": 20,
   "mute": false,
   "timeZone": "America/Los_Angeles",
   "textToSpeech": true,
   "screenSaver": true,
   "screenSaverTimeout": 600,
   "personalizedAds": true,
   "highContrastText": false,
   "identifierForAdvertising": "38400000-8cf0-11bd-b23e-10b96e40000d"
}
```

#### Operation: Setting system settings

*Operation model: Request / Response*

This operation updates a discrete setting on the device.

##### Request topic

`dab/[DEVICE_ID]/system/settings/set`

##### Request format

```typescript
interface SetSystemSettingsRequest extends DabRequest {
   [system_setting_key: keyof SystemSettings]: [value: any];
}
```

##### Sample request

```json
{
   "outputResolution": {
      "width": 3840,
      "height": 2160,
      "frequency": 60
   }
}
```

##### Response format

```typescript
interface SetSystemSettingsResponse extends DabResponse {
   [system_setting_key: keyof SystemSettings]: [value: any];
}
```

##### Sample response

Success:

```json
{
   "status": 200,
   "outputResolution": {
      "width": 3840,
      "height": 2160,
      "frequency": 60
   }
}
```

Error:

```json
{
   "status": 400,
   "error": "Setting outputResolution does not support value { width: 3840, height: 2160, frequency: 60}"
}
```

#### Operation: Start Collecting System Logs

*Operation model: Request / Response*

This operation indicates the device to start accumulating logs from running
system and user applications.

The following log categories are expected to be stored:

1. **System Logs:** Logs emitted from system services, like A/V Decoder,
   Power Manager, and Networking Module.
2. **Application Logs:** Logs emitted from apps, like Prime Video, YouTube,
   and Netflix.
3. **Crash Logs:** Dumps and logs generated when either system services or
   applications crash

Logs must be returned when log collection is completed with
`system/logs/stop-collection`, described below.

##### Request topic

`dab/[DEVICE_ID]/system/logs/start-collection`

##### Request format

```typescript
interface StartSystemLogCollectionRequest extends DabRequest {
}
```

##### Response format

```typescript
interface StartSystemLogCollectionResponse extends DabResponse {
}
```

##### Sample response

Success:

```json
{
   "status": 200
}
```

Error:

```json
{
   "status": 500,
   "error": "No more disk space, cannot begin system log collection."
}
```

#### Operation: Stop Collecting System Logs

*Operation model: Request / Response*

This operation indicates the device to stop accumulating logs and respond with
the collected logs so far.

Devices must package the following categories of logs into a `.tar.gz` file with
the following folder structure.

```
📦logs.tar.gz
 ┣ 📂system
 ┃ ┣ 📜av_decoder.txt
 ┃ ┗ 📜...
 ┣ 📂application
 ┃ ┣ 📂netflix
 ┃ ┃ ┣ 📜netflix.log
 ┃ ┃ ┗ 📜...
 ┃ ┗ 📂dab_app_id
 ┣ 📂crash
 ┃ ┣ 📜 dump_av_trace.vmcore
 ┃ ┗ 📜...
```

The intention of the structure is defined below:

* `system`: Contains relevant system logs generated by internal devices
  services. Ex: AV Pipeline, App Manager, etc.
* `application`: Contains logs produced by applications, bucketized into folders
   titled the DAB Application Identifier. For specific log files to collect per
   application, refer to the private DAB Confluence Page shared with device and
   app partners.
* `crash`: Contains files believes to be generated as a result of a crash of a
   system service or application. May be empty if no crash occurred.

The `.tar.gz` must then be base64-encoded and included in the response as per
the following specification. Only logs generated between the start and stop
collection requests must be returned.

When the `logArchive` exceeds the MQTT broker's message size limit (e.g. 200 MB)
or the device platform's memory constraints, the response must be split into
chunks. Each chunk will contain a portion of the base64-encoded `logArchive` and
will be sent in multiple MQTT responses. The `remainingChunks` parameter will
indicate how many more chunks are expected. The responses must be sent in order,
and DAB clients will wait until all chunks are received, which occurs when
`remainingChunks` is set to 0.

##### Request topic

`dab/[DEVICE_ID]/system/logs/stop-collection`

##### Operation Latency

Max period between chunk response should be at most 2 seconds.

##### Request format

```typescript
interface StopSystemLogCollectionRequest extends DabRequest
```

##### Response format

```typescript
interface StopSystemLogCollectionResponse extends DabResponse {
    logArchive: string // Byte64 encoded string from assembled tar.gz  containing logs with aforementioned folder structure 
    remainingChunks: number // Remaining StopSystemLogCollectionResponse that are expected to form the complete logArchive 
}
```

##### Sample responses

Success:
```json
{
   "status": 200,
   "logArchive": "H4sIAAAAAAA",
   "remainingChunks": 1
}
```

```json
{
   "status": 200,
   "logArchive": "AA+3B4sA5BDF",
   "remainingChunks": 0
}
```

### 5.4. Input

#### Common key codes

Below is a list of common key codes applicable to all Input operations in this
section.

<table>
  <thead>
    <tr>
      <th>Key Code</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>KEY_POWER</td>
      <td>power on / power off the device</td>
    </tr>
    <tr>
      <td>KEY_HOME</td>
      <td>go to home screen</td>
    </tr>
    <tr>
      <td>KEY_VOLUME_UP</td>
      <td>increase the volume</td>
    </tr>
    <tr>
      <td>KEY_VOLUME_DOWN</td>
      <td>decrease the volume</td>
    </tr>
    <tr>
      <td>KEY_MUTE</td>
      <td>mute audio</td>
    </tr>
    <tr>
      <td>KEY_CHANNEL_UP</td>
      <td>next channel</td>
    </tr>
    <tr>
      <td>KEY_CHANNEL_DOWN</td>
      <td>previous channel</td>
    </tr>
    <tr>
      <td>KEY_MENU</td>
      <td>go to main menu</td>
    </tr>
    <tr>
      <td>KEY_EXIT</td>
      <td>exit from current screen</td>
    </tr>
    <tr>
      <td>KEY_INFO</td>
      <td>display program information</td>
    </tr>
    <tr>
      <td>KEY_GUIDE</td>
      <td>display guide screen</td>
    </tr>
    <tr>
      <td>KEY_CAPTIONS</td>
      <td>toggle closed captions/subtitles</td>
    </tr>
    <tr>
      <td>KEY_UP</td>
      <td>navigation: go up</td>
    </tr>
    <tr>
      <td>KEY_PAGE_UP</td>
      <td>navigation: page up</td>
    </tr>
    <tr>
      <td>KEY_PAGE_DOWN</td>
      <td>navigation: page down</td>
    </tr>
    <tr>
      <td>KEY_RIGHT</td>
      <td>navigation: go right</td>
    </tr>
    <tr>
      <td>KEY_DOWN</td>
      <td>navigation: go down</td>
    </tr>
    <tr>
      <td>KEY_LEFT</td>
      <td>navigation: go left</td>
    </tr>
    <tr>
      <td>KEY_ENTER</td>
      <td>confirm current selection</td>
    </tr>
    <tr>
      <td>KEY_BACK</td>
      <td>go to the previous screen</td>
    </tr>
    <tr>
      <td>KEY_PLAY</td>
      <td>start playback</td>
    </tr>
    <tr>
      <td>KEY_PLAY_PAUSE</td>
      <td>start or pause playback</td>
    </tr>
    <tr>
      <td>KEY_PAUSE</td>
      <td>pause playback</td>
    </tr>
    <tr>
      <td>KEY_RECORD</td>
      <td>start recording</td>
    </tr>
    <tr>
      <td>KEY_STOP</td>
      <td>stop recording</td>
    </tr>
    <tr>
      <td>KEY_REWIND</td>
      <td>rewind playback (continuous)</td>
    </tr>
    <tr>
      <td>KEY_FAST_FORWARD</td>
      <td>fast forward playback (continuous)</td>
    </tr>
    <tr>
      <td>KEY_SKIP_REWIND</td>
      <td>rewind/skip (set increments)</td>
    </tr>
    <tr>
      <td>KEY_SKIP_FAST_FORWARD</td>
      <td>fast forward (set increments)</td>
    </tr>
    <tr>
      <td>KEY_0</td>
      <td>0 numeric key</td>
    </tr>
    <tr>
      <td>KEY_1</td>
      <td>1 numeric key</td>
    </tr>
    <tr>
      <td>KEY_2</td>
      <td>2 numeric key</td>
    </tr>
    <tr>
      <td>KEY_3</td>
      <td>3 numeric key</td>
    </tr>
    <tr>
      <td>KEY_4</td>
      <td>4 numeric key</td>
    </tr>
    <tr>
      <td>KEY_5</td>
      <td>5 numeric key</td>
    </tr>
    <tr>
      <td>KEY_6</td>
      <td>6 numeric key</td>
    </tr>
    <tr>
      <td>KEY_7</td>
      <td>7 numeric key</td>
    </tr>
    <tr>
      <td>KEY_8</td>
      <td>8 numeric key</td>
    </tr>
    <tr>
      <td>KEY_9</td>
      <td>9 numeric key</td>
    </tr>
    <tr>
      <td>KEY_RED</td>
      <td>red function button</td>
    </tr>
    <tr>
      <td>KEY_GREEN</td>
      <td>green function button</td>
    </tr>
    <tr>
      <td>KEY_YELLOW</td>
      <td>yellow function button</td>
    </tr>
    <tr>
      <td>KEY_BLUE</td>
      <td>blue function button</td>
    </tr>
  </tbody>
</table>

If an action is available on the device, it must be mapped to one of the
specified key codes.

#### Custom keys

The device may choose to accept key codes not specified in the Common key codes
section if they are supported (such as vendor buttons). Custom keys must start
with `KEY_CUSTOM_` prefix.

#### Operation: Get available input keys

*Operation model: Request / Response*

Retrieve the list of input key codes supported by the device.

##### Request topic

`dab/[DEVICE_ID]/input/key/list`

##### Request format

```typescript
type ListSupportedKeysRequest = DabRequest
```

##### Response format

```typescript
interface ListSupportedKeysResponse extends DabResponse {
   keyCodes: string [];
}
```

##### Response parameters

<table>
  <thead>
    <tr>
      <th>Parameter</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>keyCodes</td>
      <td>list of key codes supported by the device</td>
    </tr>
  </tbody>
</table>

##### Sample response

```json
{
   "status": 200,
   "keyCodes": [
      "KEY_POWER",
      "KEY_HOME",
      "KEY_MENU",
      "KEY_EXIT",
      "KEY_INFO",
      "KEY_UP",
      "KEY_RIGHT",
      "KEY_DOWN",
      "KEY_LEFT",
      "KEY_ENTER",
      "KEY_BACK"
   ]
}
```

#### Operation: Key press

*Operation model: Request / Response*

This operation sends a key press to the device. Key press is an action that can
be associated with the key press on the remote control. The operation mimics the
same behavior as a key press and release of a remote control. It represents a
button name or function name typically found on the remote control.

#### Request topic

`dab/[DEVICE_ID]/input/key-press`

##### Request format

```typescript
interface SendKeyPressRequest extends DabRequest {
   keyCode: string;
}
```

##### Request Parameters

<table>
  <thead>
    <tr>
      <th>Parameter</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>keyCode</td>
      <td>string literal, prefixed with <code>KEY_</code>, representing the button name / function name, e.g. <code>KEY_PLAY</code>. A valid key code consists of a sequence of alpha-numeric characters and/or <code>_</code> (underscore) characters. The minimum number of characters is 1 and the maximum is 64.</td>
    </tr>
  </tbody>
</table>

##### Sample requests

To navigate left in the menu:

```json
{ "keyCode": "KEY_LEFT" }
```

To press `Vendor 1` key on the remote. `Vendor 1` is a custom key and its key
code starts with `KEY_CUSTOM_`:

```json
{ "keyCode": "KEY_CUSTOM_VENDOR_1" }
```

##### Response format

```typescript
type SendKeyPressResponse = DabResponse
```

#### Operation: Long press

*Operation model: Request / Response*

This operation mimics the action of pressing a key for a specified duration.

##### Request topic

`dab/[DEVICE_ID]/input/long-key-press`

##### Operation latency

The operation MUST be completed within 200 milliseconds plus the duration of the keypress. 

##### Request format

```typescript
interface SendLongKeyPressRequest extends DabRequest {
   keyCode: string;
   durationMs: int;
}
```

##### Request parameters

<table>
  <thead>
    <tr>
      <th>Parameter</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>keyCode</td>
      <td>string literal, prefixed with <code>KEY_</code>, representing the button name / function name, e.g. <code>KEY_PLAY</code>. A valid key code consists of a sequence of alpha-numeric characters and/or <code>_</code> (underscore) characters. The minimum number of characters is 1 and the maximum is 64.</td>
    </tr>
    <tr>
      <td>durationMs</td>
      <td>Duration in milliseconds for which the key specified in keyCode should be pressed.</td>
    </tr>
  </tbody>
</table>

##### Sample request

To press the left key for 3 seconds

```json
{ "keyCode": "KEY_LEFT", "durationMs": 3000 }
```

##### Response format

```typescript
type SendLongKeyPressResponse = DabResponse
```

### 5.5. Output

Retrieves outputs from the device (such as images, audio, etc.). Images are the
only outputs currently supported.

#### Operation: Capture image

*Operation model: Request / Response*

This operation captures the current output of the device and returns an image
that is base64 encoded. The image must include both graphics plane content and
clear (non-DRM) stream video. DRM content and any video that passes through the
device's secure video pipeline or trusted execution environment shall not be in
the captured image. The image data must be in PNG format.

##### Request topic

`dab/[DEVICE_ID]/output/image`

##### Operation Latency

The operation MUST be completed within 2 seconds.

##### Request format

```typescript
type CaptureScreenshotRequest = DabRequest
```

##### Response format

```typescript
interface CaptureScreenshotResponse extends DabResponse {
   outputImage: string;
}
```

##### Response parameters

<table>
  <thead>
    <tr>
      <th>Parameter</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>outputImage</td>
      <td>base64 encoded PNG image in a <a href="https://www.rfc-editor.org/rfc/rfc2397">data URL</a></td>
    </tr>
  </tbody>
</table>

##### Sample response

```json
{
   "outputImage": "data:image/png;base64,iVBORw0KGgo..."
}
```

### 5.6. Device and Application Telemetry

Device and application telemetry lets clients gather device and application
metrics. To start or stop device or application telemetry, send a request to the
correct topic.

Once started, the device publishes metrics to the notification topic until
stopped. Telemetry is stateless; request it again after a restart.

#### Operation: Stop Device Telemetry

*Operation model: Request / Response*

##### Request topic

`dab/[DEVICE_ID]/device-telemetry/start`

##### Request format

```typescript
interface StartDeviceTelemetryRequest extends DabRequest {
   duration: number;
}
```

##### Request parameters

<table>
  <thead>
    <tr>
      <th>Parameter</th>
      <th>Default</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>duration</td>
      <td>-</td>
      <td>desired telemetry update duration (milliseconds)</td>
    </tr>
  </tbody>
</table>

##### Sample request

Request to start device telemetry with a 500 millisecond pull duration:

```json
{
   "duration": 500
}
```

##### Response format

```typescript
interface StartDeviceTelemetryResponse extends DabResponse {
   duration: number;
}
```

##### Response parameters

<table>
  <thead>
    <tr>
      <th>Parameter</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>duration</td>
      <td>actual pull duration the device supports for telemetry updates</td>
    </tr>
  </tbody>
</table>

##### Sample response

Telemetry started with an actual duration of 1 second:

```json
{
   "status": 200,
   "duration": 1000
}
```

#### Operation: Stop Device Telemetry

*Operation model: Request / Response*

##### Request topic

`dab/[DEVICE_ID]/device-telemetry/stop`

##### Request format

```typescript
interface StopDeviceTelemetryRequest extends DabRequest {
}
```

##### Response format

```typescript
type StopDeviceTelemetryResponse = DabResponse
```

### Operation: Start Application Telemetry

*Operation model: Request / Response*

##### Request topic

`dab/[DEVICE_ID]/app-telemetry/start`

##### Operation latency

The operation MUST be completed within 700 milliseconds.

##### Request format

```typescript
interface StartApplicationTelemetryRequest extends DabRequest {
   appId: string;
   duration: number;
}
```

##### Request parameters

<table>
  <thead>
    <tr>
      <th>Parameter</th>
      <th>Default</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>appId</td>
      <td>-</td>
      <td>application identifier</td>
    </tr>
    <tr>
      <td>duration</td>
      <td>-</td>
      <td>desired telemetry update duration (milliseconds)</td>
    </tr>
  </tbody>
</table>

##### Sample request

A request to start telemetry for the application `media-player-1` with the duration of 5 seconds
```json
{
   "appId": "media-player-1",
   "duration": 5000
}
```

##### Response format

```typescript
interface StartApplicationTelemetryResponse extends DabResponse {
   duration: number;
}
```

##### Response parameters

<table>
  <thead>
    <tr>
      <th>Parameter</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>duration</td>
      <td>actual duration device is capable of for telemetry updates</td>
    </tr>
  </tbody>
</table>

##### Sample response

Telemetry started with the actual duration of 1 second
```json
{
   "status": 200,
   "duration": 1000
}
```

#### Operation: Stop Application Telemetry

*Operation model: Request / Response*

##### Request topic

``dab/[DEVICE_ID]/app-telemetry/stop``

##### Request format

```typescript
interface StopApplicationTelemetryRequest extends DabRequest {
   appId: string;
}
```

##### Request parameters

<table>
  <thead>
    <tr>
      <th>Parameter</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>appId</td>
      <td>Application identifier</td>
    </tr>
  </tbody>
</table>

##### Sample request

To stop the telemetry for the application `media-player-2`:
```json
{
   "appId": "media-player-2"
}
```

##### Response format

```typescript
type StopApplicationTelemetryResponse = DabResponse
```

##### Telemetry notification topics

<table>
  <thead>
    <tr>
      <th>Scope</th>
      <th>Notification Topic</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Device</td>
      <td><code>dab/[DEVICE_ID]/device-telemetry/metrics</code></td>
      <td>-</td>
    </tr>
    <tr>
      <td>Application</td>
      <td><code>dab/[DEVICE_ID]/app-telemetry/metrics/[APP_ID]</code></td>
      <td><code>[APP_ID]</code> is the application ID</td>
    </tr>
  </tbody>
</table>

##### Example

The application identified in the system by `media-player-1` will have its
metrics delivered to the notification topic
`dab/app-telemetry/metrics/media-player-1`

### Telemetry metrics

List of supported metrics.

<table>
  <thead>
    <tr>
      <th>Metric</th>
      <th>Unit</th>
      <th>Description</th>
      <th>Example</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>cpu</td>
      <td>percentage expressed as anumber</td>
      <td>Overall CPU usage scaled to the range of 0% to 100%</td>
      <td>26</td>
    </tr>
    <tr>
      <td>memory</td>
      <td>kilobytes</td>
      <td><em>optional</em> Memory usage</td>
      <td>43008</td>
    </tr>
  </tbody>
</table>

If an application is not running, but telemetry collection is active, the device
should return 0 for both CPU and memory.

### Message format

```typescript
interface TelemetryMessage {
   timestamp: unix_timestamp_ms;
   metric: string;
   value: number;
}
```

### Message parameters

<table>
  <thead>
    <tr>
      <th>Parameter</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>timestamp</td>
      <td>UNIX Epoch time in milliseconds</td>
    </tr>
    <tr>
      <td>metric</td>
      <td>metric name</td>
    </tr>
    <tr>
      <td>value</td>
      <td>value of the metric</td>
    </tr>
  </tbody>
</table>

### Sample messages

```json
{
   "timestamp": 1568815387403,
   "metric": "cpu",
   "value": 33.5
}
```

```json
{
   "timestamp": 1568815390414,
   "metric": "memory",
   "value": 44040192
}
```

## 5.7. Health Check

### Operation: Run a health check

*Operation model: Request / Response*

This operation checks whether DAB is enabled and operational on the device.

#### Request topic

`dab/[DEVICE_ID]/health-check/get`

#### Request format

```typescript
type CheckDeviceHealthRequest = DabRequest
```

#### Response format

```typescript
interface CheckDeviceHealthResponse extends DabResponse {
   healthy: boolean;
   message?: string;
}
```

#### Response parameters

<table>
  <thead>
    <tr>
      <th>Parameter</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>healthy</td>
      <td><code>true</code> when DAB is enabled and operational, <code>false</code> otherwise.</td>
    </tr>
    <tr>
      <td>message</td>
      <td><em>optional</em> Message to indicate any problems with the DAB protocol.</td>
    </tr>
  </tbody>
</table>

#### Sample responses

```json
{
   "status": 200,
   "healthy": true
}
```

```json
{
   "status": 200,
   "healthy": false,
   "message": "Internal DAB error"
}
```

## 5.8 *Optional* General Notifications

The device may publish errors, warnings, or general information to the
notification topic. These messages provide developers internal observability
into DAB's state and help debug potential platform issues. Although not
required, providing retained MQTT messages that inform clients of DAB's state is
highly recommended (see examples).

### Notification topic

`dab/[DEVICE_ID]/messages`

### Notification format

```typescript
type NotificationLevel = "info" | "warn" | "debug" | "trace" | "error";

interface Notification {
   timestamp: unix_timestamp_ms;
   level: NotificationLevel;
   message: string;
}
```

### Notification levels

<table>
  <thead>
    <tr>
      <th>Notification level</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><code>info</code></td>
      <td>A general message.</td>
    </tr>
    <tr>
      <td><code>warn</code></td>
      <td>A message indicating a warning, potentially problematic event.</td>
    </tr>
    <tr>
      <td><code>debug</code></td>
      <td>A message intended to help with the debugging of the problem.</td>
    </tr>
    <tr>
      <td><code>trace</code></td>
      <td>A detailed, low-level message.</td>
    </tr>
    <tr>
      <td><code>error</code></td>
      <td>A message indicating an error event.</td>
    </tr>
  </tbody>
</table>

### Notification parameters

<table>
  <thead>
    <tr>
      <th>Parameter</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>timestamp</td>
      <td>UNIX Epoch time in milliseconds</td>
    </tr>
    <tr>
      <td>level</td>
      <td>Notification level</td>
    </tr>
    <tr>
      <td>message</td>
      <td>Message</td>
    </tr>
  </tbody>
</table>
#### Suggested notifications

```json
{
   "timestamp": 1610704569349,
   "level": "info",
   "message": "DAB started successfully and is online."
}
```

```json
{
   "timestamp": 1610704569349,
   "level": "error",
   "message": "DAB faced an exception. Terminating abnormally."
}
```

## 5.9. Voice

This section applies to all devices where users may interact with the device
using a microphone.

### Voice Systems

A voice system is a platform assistant or speech recognizer that uses audio
input.

### Operation: Listing all available voice systems

*Operation model: Request / Response*

This operation lists the available voice systems installed on the device.

#### Request topic

`dab/[DEVICE_ID]/voice/list`

#### Request format

```typescript
type ListSupportedVoiceSystemsRequest = DabRequest
```

#### Response format

```typescript
interface VoiceSystem {
   name: string;
   enabled: boolean;
}
```

<table>
  <thead>
    <tr>
      <th>Parameter</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>name</td>
      <td>Name of the voice system</td>
    </tr>
    <tr>
      <td>enabled</td>
      <td>'true' when the voice system is enabled on the platform, 'false' otherwise.</td>
    </tr>
  </tbody>
</table>

```typescript
interface ListSupportedVoiceSystemsResponse extends DabResponse {
   voiceSystems: VoiceSystem [];
}
```

#### Response parameters

<table>
  <thead>
    <tr>
      <th>Parameter</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>voiceSystems</td>
      <td>Available <a href="#voice-system-name-registry">voice systems</a> supported by the device.</td>
    </tr>
  </tbody>
</table>

#### Sample response

```json
{
   "status": 200,
   "voiceSystems": [
      {
         "name": "AmazonAlexa",
         "enabled": true
      },
      {
         "name": "GoogleAssistant",
         "enabled": false
      }
   ]
}
```

### Operation: Set enabled voice systems

*Operation model: Request / Response*

This operation changes the enabled voice system(s). The platform decides if
multiple systems can run at once.

#### Request topic

`dab/[DEVICE_ID]/voice/set`

#### Request format

```typescript
interface SetVoiceSystemRequest extends DabRequest {
   voiceSystem: VoiceSystem;
}
```

#### Sample request

```json
{
   "voiceSystem": {
      "name": "GoogleAssistant",
      "enabled": true
   },
}
```

#### Response format

```typescript
interface SetVoiceSystemResponse extends DabResponse {
   voiceSystem: VoiceSystem;
}
```

#### Sample response

Success:

```json
{
   "status": 200,
   "voiceSystem": {
      "name": "GoogleAssistant",
      "enabled": true
   },
}
```

Error:

```json
{
   "status": 400,
   "error": "Setting voiceSystem failed"
}
```

### Operation: Send audio to voice system

*Operation model: Request / Response*

This operation provides an HTTP URL. The URL identifies an audio file of MIME
type `audio/wav`. This file is used for a voice request. The platform must issue
a GET request to the URL. The platform routes the audio. It should treat the
audio as if it came from the device's microphone.

The audio must be a WAV format file containing 16-bit linear PCM, 16-kHz sample
rate, single channel, little-endian byte order format. The sample rate must be
contained in the header.

File size must not exceed 4MB.

**Note:** The voice assistant language is expected to match the system language. 
In typical usage, a system language change should be performed before sending 
any voice command via audio file to ensure proper language recognition.

#### Request topic

`dab/[DEVICE_ID]/voice/send-audio`

#### Request format

```typescript
interface SendVoiceAsAudioRequest extends DabRequest {
   fileLocation: url;
   voiceSystem?: string;
}
```

#### Request parameters

<table>
  <thead>
    <tr>
      <th>Parameter</th>
      <th>Default</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>fileLocation</td>
      <td>-</td>
      <td>HTTP URL identifying the location of the audio resource for the platform to download.</td>
    </tr>
    <tr>
      <td>voiceSystem</td>
      <td>-</td>
      <td><em>Optional</em> Voice system to direct the audio to. The voice system must be enabled first. As <a href="#voice-system-name-registry">defined in the DAB Voice System Registry</a></td>
    </tr>
  </tbody>
</table>

#### Sample request

```json
{
   "fileLocation": "http://192.168.1.8:80/utterance_1.wav",
   "voiceSystem": "AmazonAlexa"
}
```

#### Response format

```typescript
type SendVoiceAsAudioResponse = DabResponse
```

#### Sample responses

```json
{
   "status": 200
}
```

```json
{
   "status": 400,
   "error": "File not available at fileLocation"
}
```

### Operation: Send text to voice system

*Operation model: Request / Response*

This operation provides a text string to be used for a simulated voice request.
The platform must treat the text string as if it were the transcribed output of
a traditional, microphone-initiated voice request.

#### Request topic

`dab/[DEVICE_ID]/voice/send-text`

#### Request format

```typescript
interface SendVoiceAsTextRequest extends DabRequest {
   requestText: string;
   voiceSystem: string;
}
```

#### Request Parameters

<table>
  <thead>
    <tr>
      <th>Parameter</th>
      <th>Default</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>requestText</td>
      <td>-</td>
      <td>The string to be passed as output of a voice system.</td>
    </tr>
    <tr>
      <td>voiceSystem</td>
      <td>-</td>
      <td>Voice system to direct the audio to. The voice system must be enabled first.</td>
    </tr>
  </tbody>
</table>

#### Sample request

```json
{
   "requestText": "Play Ed Sheeran on YouTube",
   "voiceSystem": "GoogleAssistant"
}
```

#### Response format

```typescript
type SendVoiceAsTextResponse = DabResponse
```

#### Sample responses

```json
{
   "status": 200
}
```

```json
{
   "status": 501,
   "error": "Not implemented"
}
```

## 5.10. Content

The following operations allow retrieval and consumption of content, like
movies and TV shows, available on the device.

Content is associated with an application that can open it.
If the same content can be opened by multiple applications, there will be a
separate `ContentEntry` for each application, each with its own unique entry ID.

The operations must not change what is displayed on the screen.

```typescript
interface ContentEntry {
   // A unique identifier of the content.
   entryId: string,
   // The identifier of the application that opens the content.
   appId: string,
   // The title of the content. The language of the title should match the device's settings.
   title: string,
   // Base64-encoded PNG image of the content's poster in a data URL format.
   poster: string,
   // The list of category of the content.
   categories: ContentCategory[],
}

enum ContentCategory {
   ContinueWatching = "ContinueWatching",
   Movies = "Movies",
   TvShows = "TvShows",
   LiveEvents = "LiveEvents",
   LinearTv = "LinearTv",
   Video = "Video", // as social media and online video
   Trending = "Trending",
   Others = "Others",
}
```

### Operation: Search content

*Operation model: Request / Response*

This operation searches for content available on the device.
The operation mimics a user performing a search in the device's search menu.

#### Request topic

`dab/[DEVICE_ID]/content/search`

#### Request format

```typescript
interface SearchContentRequest extends DabRequest {
   searchText: string;
}
```

#### Request Parameters

<table>
  <thead>
    <tr>
      <th>Parameter</th>
      <th>Default</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>searchText</td>
      <td>N/A</td>
      <td>The string to be used for the search.</td>
    </tr>
  </tbody>
</table>

#### Sample request

```json
{
   "searchText": "home",
}
```

#### Response format

```typescript
interface SearchContentResponse extends DabResponse {
   entries: ContentEntry[],
}
```

#### Sample responses

```json
{
   "status": 200,
   "entries": [
      {
         "entryId": "65850a53-3cf1-4b0e-bc02-d015fa9bec77",
         "title": "Home Alone",
         "appId": "PrimeVideo",
         "poster":"data:image/png;base64, iVBsRw0KdGgoAAAANSUhE...",
         "categories": ["Movies"],
      },
      {
         "entryId": "b8884dfa-60e8-448a-99f3-bcf283dcc903",
         "title": "Home Alone",
         "appId": "YouTube",
         "poster":"data:image/png;base64, iVBsRw0KdGgoAAAANSUhE...",
         "categories": ["Movies", "ContinueWatching"],
      },
      {
         "entryId": "a7eb554a-27bd-40e4-825f-cc7ac90fb6b2",
         "title": "I Don't Feel at Home in This World Anymore",
         "appId": "Netflix",
         "poster":"data:image/png;base64, iVBO9zQrdWqqtSDa...",
         "categories": ["Movies"],
      },
   ]
}
```

### Operation: Content recommendations

*Operation model: Request / Response*

This operation retrieves content recommendations from the device.
The returned content must match the content displayed in the device's
recommendations menu.

#### Request topic

`dab/[DEVICE_ID]/content/recommendations`

#### Request format

```typescript
type ListRecommendationsContentRequest = DabRequest;
```

#### Response format

```typescript
interface ListRecommendationsResponse extends DabResponse {
   entries: ContentEntry[],
}
```

#### Sample responses

```json
{
   "status": 200,
    "entries": [
      {
         "entryId": "65850a53-3cf1-4b0e-bc02-d015fa9bec77",
         "title": "Home Alone",
         "appId": "PrimeVideo",
         "poster":"data:image/png;base64, iVBsRw0KdGgoAAAANSUhE...",
         "categories": ["Movies"],
      },
      {
         "entryId": "b8884dfa-60e8-448a-99f3-bcf283dcc903",
         "title": "Home Alone",
         "appId": "YouTube",
         "poster":"data:image/png;base64, iVBsRw0KdGgoAAAANSUhE...",
         "categories": ["Movies", "ContinueWatching"],
      },
      {
         "entryId": "a7eb554a-27bd-40e4-825f-cc7ac90fb6b2",
         "title": "I Don't Feel at Home in This World Anymore",
         "appId": "Netflix",
         "poster":"data:image/png;base64, iVBO9zQrdWqqtSDa...",
         "categories": ["Movies"],
      },
   ]
}
```

### Operation: Open content

*Operation model: Request / Response*

This operation opens the specified content and returns only after the
application has started.

It mimics a user clicking on the content tile in search results or
recommendations.

Note: It differs from the launch application with content ID operation, where
content ID is an application specific content ID.

#### Request topic

`dab/[DEVICE_ID]/content/open`

#### Request format

```typescript
interface OpenContentRequest extends DabRequest {
   entryId: string,
}
```

#### Request Parameters

<table>
  <thead>
    <tr>
      <th>Parameter</th>
      <th>Default</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>entryId</td>
      <td>N/A</td>
      <td>The ID of the content.</td>
    </tr>
  </tbody>
</table>

#### Sample request

```json
{
   "entryId": "homealone123"
}
```

#### Response format

```typescript
type OpenContentResponse = DabResponse;
```

#### Sample responses

Success:
```json
{
   "status": 200
}
```

Error:
```json
{
   "status": 404,
   "error": "EntryID 'homealone123' not found."
}
```

### Operation: Discover existing active device

*Operation model: Request / Response*

This operation provides a way for user client to discover devices under the same
MQTT broker. For each DAB capable device a device discovery response must be
provided.

#### Request topic

`dab/discovery`

#### Request format

```typescript
type DiscoverDevicesRequest = DabRequest
```

#### Response format

```typescript
interface DiscoverDevicesResponse extends DabResponse {
   ip: string;
   deviceId: string;
}
```

#### Response parameters

<table>
  <thead>
    <tr>
      <th>Parameter</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>ip</td>
      <td>IP Address of the device.</td>
    </tr>
    <tr>
      <td>deviceId</td>
      <td>The unique identifier of this device. Should be same as DEVICE_ID in other topics.</td>
    </tr>
  </tbody>
</table>

#### Sample responses

```json
{
   "status": 200,
   "ip": "192.168.1.119",
   "deviceId": "device3"
}
```

## 6. Versioning of the protocol

*Operation model: Request / Response*

This operation lets the user clients know the DAB version implemented on a
device client. A device may implement multiple DAB protocol versions.

### Operation: Get Protocol Version

A protocol version consists of the major version and the minor version delimited
by a full stop character `.` The major and minor versions are non-negative
integers.

Examples:

Valid versions: `1.0`, `2.3`, `0.48`

Invalid versions `1.a`, `x.5`

#### Request Topic

`dab/[DEVICE_ID]/version`

#### Request Format

```typescript
type GetDabVersionRequest = DabRequest
```

#### Response Format

```typescript
interface GetDabVersionResponse extends DabResponse {
   versions: string [];
}
```

#### Response Parameters

<table>
  <thead>
    <tr>
      <th>Parameter</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>versions</td>
      <td>a list of versions of the protocol implemented on the device</td>
    </tr>
  </tbody>
</table>

#### Sample Response

```json
{
   "status": 200,
   "versions": ["2.1"]
}
```

## 7. Appendix

### Application ID Registry

<table>
  <thead>
    <tr>
      <th>Application</th>
      <th>DAB appId</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Netflix</td>
      <td>Netflix</td>
    </tr>
    <tr>
      <td>YouTube</td>
      <td>YouTube</td>
    </tr>
    <tr>
      <td>Amazon Prime Video</td>
      <td>PrimeVideo</td>
    </tr>
    <tr>
      <td>BBC iPlayer</td>
      <td>uk.co.bbc.iplayer</td>
    </tr>
    <tr>
      <td>YouTube TV</td>
      <td>YouTubeTV</td>
    </tr>
  </tbody>
</table>

**Note:** This registry provides common application identifiers for well-known applications. However, additional appIDs beyond those listed in this registry are allowed, particularly to enable development and testing of applications. This is especially relevant for sideloading scenarios where developers may use custom or development versions of applications.

### Voice System Name Registry

<table>
  <thead>
    <tr>
      <th>Voice System</th>
      <th>DAB Voice System Name</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Google Assistant</td>
      <td>GoogleAssistant</td>
    </tr>
    <tr>
      <td>Amazon Alexa</td>
      <td>AmazonAlexa</td>
    </tr>
    <tr>
      <td>LG ThinQ</td>
      <td>LGThinQ</td>
    </tr>
    <tr>
      <td>Yandex Alice</td>
      <td>YandexAlice</td>
    </tr>
    <tr>
      <td>TiVo Conversation</td>
      <td>TiVoConversation</td>
    </tr>
    <tr>
      <td>WhaleAIVoice</td>
      <td>WhaleAIVoice</td>
    </tr>
  </tbody>
</table>

### Process for Contributing to Registry

If you are an application or voice system owner and your identifier is not found
in the registry, you can register it by following these steps:

1. Fork this specification repository.
2. Add and commit your application or voice system identifier in your repo.
3. Create a pull request to merge your additions to this DAB specification.
4. A DAB consortium member will review your pull request, approve, and
   merge promptly.

[Learn how to create a pull request from a forked repository](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/proposing-changes-to-your-work-with-pull-requests/creating-a-pull-request-from-a-fork)
