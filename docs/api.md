# API Documentation

The purpose of this documentation is to cover how you would interact with the Orbiting API without a library.

> [!TIP]
> It is strongly recommended that if there is an official API wrapper for your preferred programming language that you use it. Below are links to a couple that were available at the time of writing this.
>
> -   [NodeJS](https://npmjs.com/package/orbiting)

## App REST API

In order to control your app there are a couple important endpoints that should be kept in mind. The base URL used for all of these endpoints should be:

```
https://orbiting.app/api
```

### Authentication

In order to use any of these endpoints you must be authenticated as the app you wish to act upon and on behalf of. To do so you must send your app token along with each request, which can be obtained on your app's auth page.

> [!IMPORTANT]
> These tokens are [JWTs](https://jwt.io/) signed by Orbiting's backend that **do not expire**.

In order to be authorized, all requests should include this header:

```http
Authorization: Bearer <token>
```

With `<token>` replaced with your app's token, of course.

### Endpoints

> [!NOTE]
> If there is an error with any endpoint it will be reflected in the **status code of the response**. Along with a JSON response object containing a top-level `error` property. If it was a validation error, the `error` property will likely be returned paired with a `message` property.

#### Get App Information

```http
GET /apps HTTP/2
```

#### Get Config

```http
GET /apps/config HTTP/2
```

This endpoint returns you the current value of the app's config compliant to the current schema.

#### Update Config

As of right now there is no way to do this. We want to discourage usage of Orbiting in this way, because as of right now our goal is to allow for on-the-fly control panels. If your app requires updating values and pushing those changes, it's likely you should be using a database.

#### Define Schema

```http
POST /apps/schema HTTP/2

{
    "type": "object",
    "properties": {
        "maxSignUps": {
            "type": "number",
            "default": 10
        }
    }
}
```

This endpoint is how you define a schema. It accepts a JSON body where the top-level object is a [JSON schema object](https://json-schema.org/understanding-json-schema/reference/object). It is recommended you follow only setting the `type` and `properties` as a lot of the other options regularly available are stripped away by the backend. This is due to them making no sense in the context of Orbiting's control panel as of right now.

On success, returns an object containing the new `config` adjusted to the schema changes, the new `configSchema` just pushed up, and `configSchemaHash`.

#### Define Layout

```http
POST /apps/layout HTTP/2

{
    "layout": [
        {
            "title": "Example",
            "description": "Hello world",
            "controls": [{ "for": "maxSignUps" }]
        }
    ]
}
```

This endpoint is how you specify the layout of an app's control panel. It accepts an array of section objects inside a top-level property `layout`.

On success, returns an object containing the new `layout`.

## Live Updates

If you want to have live updates when your app's config is updated, you should use our websocket connection. Our websocket is hosted at:

```

wss://orbiting.app/api/ws

```

Any incorrect packets sent to the websocket will result in automatic termination of the connection. This will be accompanied by an `error` packet with the `data` being the reason.

### Packet Structure

For both incoming (S -> C) and outgoing (C -> S) packets they must be **stringified** JSON objects following this structure:

```ts
interface PacketType {
    type: string
    data: unknown
}
```

### Handshake

Within the first ten seconds of connecting to the websocket your client is responsible for sending an `identify` packet containing the identifying token for your app, like so:

```json
{
    "type": "identify",
    "data": {
        "token": "..."
    }
}
```

After you have done that the server will respond with a `hello` packet containing the `heartbeartIntervalMS`.

```json
{
    "type": "hello",
    "data": {
        "heartbeatIntervalMS": 60000
    }
}
```

> [!IMPORTANT]
> Along with the `hello` packet you will also recieve a `config` packet, which contains the JSON form of your app's current config as its `data`.

### Heartbeat

With `heartbeartIntervalMS`, you now have the time in milliseconds you must regularly send a `heartbeat` packet at. The server does allow some leeway (aka jitter) to prevent disconnecting lagging connections if the packet is a little late. A heartbeat packet should be exactly this:

```json
{
    "type": "heartbeat",
    "data": {}
}
```

The server will send back a heartbeat exactly like as an acknowledgement to the packet. If your client does not recieve a heartbeat in return it should close the connection.

### Waiting for Updates

After all that is done, as long as your client is sending heartbeat packets regularly, you will recieve `config` packets whenever the panel on the website has changes pushed to it.

### Errors & Invalidation

Any errors will be provided from an `error` packet containing the reason for the error in the `data` packet. All errors as of writing this result in a disconnection of the client.

Invalidation occurs when the app is identifying app has its token changed or the app itself is deleted. Aside from termination of the connection, it will be accompanied by an `invalidate` packet.
