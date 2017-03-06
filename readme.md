
Networked A-Frame
=======

**Bringing Social VR to the Web**

Write full-featured Social VR experiences on the web, with minimal networking knowledge required.

<div>
  <a href="#features">Features</a>
  &mdash;
  <a href="#getting-started">Getting Started</a>
  &mdash;
  <a href="#more-examples">Examples</a>
  &mdash;
  <a href="#documentation">Documentation</a>
  &mdash;
  <a href="#stay-in-touch">Stay in touch</a>
</div>

Features
--------
* Includes everything you need to create multiplayer / social WebVR apps and games.
* WebRTC with no experience required. Take advantage of low-latency, peer-to-peer networking over UDP with minimal effort.
* Audio streaming to let your users talk in-app.
* Bandwidth sensitive. Only sends network updates when a synced component has changed. Option to compress network updates.
* Extendable. Sync any A-Frame component, including your own, without changing the component code at all.


Getting Started
---------------
 ```sh
git clone https://github.com/haydenjameslee/networked-aframe.git  # Clone the repository.
cd networked-aframe && npm install && npm run easyrtc-install  # Install dependencies.
npm start  # Start the local development server.
```
With the server running, browse the examples at `http://localhost:8080`. Open another browser tab and point it to the same URL to see the other client.


Basic Example
-------------
```html
<html>
  <head>
    <title>My Networked A-Frame Scene</title>
    <script src="https://aframe.io/releases/0.5.0/aframe.min.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/socket.io/1.4.5/socket.io.min.js"></script>
    <script src="easyrtc/easyrtc.js"></script>
    <script src="xxx/networked-aframe.min.js"></script>
    <script>
      function onConnect () {
        naf.entities.createAvatar('#avatar-template', '0 1.6 0', '0 0 0');
      }
    </script>
  </head>
  <body>
    <a-scene network-scene>
      <a-assets>
        <script id="avatar-template" type="text/html">
          <a-entity class="avatar">
            <a-box class="head"
              color="#5985ff"
              scale="0.45 0.5 0.4"
            ></a-sphere>
          </a-entity>
        </script>
      </a-assets>
    </a-scene>
  </body>
</html>
```

More Examples
-------------

Open in two tabs if nobody else is online.

#### Simple

[http://haydenlee.io/networked-aframe/basic.html](http://haydenlee.io/networked-aframe/basic.html)

#### Dance Party!

[http://haydenlee.io/networked-aframe/dance-party.html](http://haydenlee.io/networked-aframe/dance-party.html)


Documentation
-------------

### Overview

Networked A-Frame works by syncing entities and their components to other connected clients. To connect to a room you need to add the [`network-scene` component](#scene-component) to the A-Frame a-scene element. For an entity to be synced, you need to create a [networked entity](#creating-networked-entities). By default the `position` and `rotation` components are synced, but if you want to sync other components or child components you need to define a [schema](#syncing-custom-components). For more advanced control over the WebRTC datachannel see the section on [Broadcasting Custom Messages](#broadcasting-custom-messages) and [Settings](#settings).


### Scene component

Required on the A-Frame `<a-scene>` component.

```html
<a-scene network-scene="
  app: <appId>;
  room: <roomName>;
  audio: false;
  debug: false;
  onConnect: onConnect;
  connectOnLoad: true;
  signallingUrl: /;
">
  ...
</a-scene>
```

| Property | Description | Default Value |
| -------- | ----------- | ------------- |
| app  | Unique app name | default |
| room  | Unique room name. Can be multiple per app | default |
| audio  | Turn on / off microphone audio streaming for your app | false |
| debug  | Turn on / off Networked A-Frame debug logs | false |
| onConnect  | Function to be called when client has successfully connected to the server | onConnect |
| connectOnLoad  | Connect to the server as soon as the webpage loads  | true |
| signallingUrl  | Customize where the signalling server is located | / |


### Creating Networked Entities

```javascript
naf.entities.createAvatar(template, position, rotation)
```

Create an avatar that follows your camera's movements. Should only be called once. The avatar is hidden for you but visible for other players.

```javascript

naf.entities.createNetworkEntity(template, position, rotation)
```
Create an instance of a template to be synced across clients. The position and rotation will be synced by default. The [`aframe-lerp-component`](https://github.com/haydenjameslee/aframe-lerp-component) is added to allow for less network updates while keeping smooth motion.

| Parameter | Description
| -------- | -----------
| template  | A css selector to a script tag stored in `<a-assets>` - [Template documentation](https://github.com/ngokevin/kframe/tree/master/components/template)
| position  | An A-Frame position string for the initial position of the entity, eg. '0 0 0'
| rotation  | An A-Frame rotation string for the initial rotation of the entity, eg '0 45 0'


### Syncing Custom Components

By default, the A-Frame `position` and `rotation` components on the root entity are synced when a network entity is created.

To sync other components and components of child entities you need to define a schema per template. Here's how to define and add a schema:

```javascript
var avatarSchema = {
  template: '#avatar-template',
  components: [
    'position',
    'rotation',
    'scale',
    {
      selector: '.head',
      component: 'material'
    },
    {
      selector: '.hairs',
      component: 'show-child'
    }
  ]
};
naf.schemas.add(avatarSchema);
```

Components of the root entity can be defined with the name of the component. Components of child entities can be defined with an object with both the `selector` field, which uses a standard CSS selector to be used by `document.querySelector`, and the `component` field which specifies the name of the component.

Once you've defined the schema then add it to the list of schemas by calling `naf.schemas.add(YOUR_SCHEMA)`.

Component data is retrieved by the A-Frame `getData` function. During the network tick each component's data is checked against its previous synced value; if the data object has changed at all it will be synced across the network.


### Broadcasting Custom Messages

```javascript
naf.connection.subscribeToDataChannel(dataType, callback)
naf.connection.unsubscribeToDataChannel(dataType)
naf.connection.broadcastData(dataType, data)
naf.connection.broadcastDataGuaranteed(dataType, data)
```

Subscribe and unsubscribe callbacks to network messages specified by `dataType`. Send messages to other clients with the `broadcastData` functions.

`broadcastData` messages are sent P2P using UDP and are not guaranteed to make it to other clients (although they will most of the time, [see why](https://en.wikipedia.org/wiki/User_Datagram_Protocol)). `broadcastDataGuaranteed` messages are currently sent via the websocket connection to the server using TCP, and hence not using WebRTC at all. These messages are guaranteed to be delivered to all connected clients. In the future a reliable protocol may be added on top of UDP instead of relying on the TCP websocket connection.

| Parameter | Description
| -------- | -----------
| dataType  | String to identify a network message. `u` is a reserved data type, don't use it pls
| callback  | Function to be called when message of type `dataType` is received.
| data | Object to be sent to all other clients


### Settings

```javascript
naf.globals.updateRate
```

Frequency the network component `sync` function is called, per second. 10-20 is normal for most Social VR applications. Default is `15`.

```javascript
naf.globals.compressSyncPackets
```

Compress each sync packet into a minimized but harder to read JSON object for saving bandwidth. Default is `false`.

To measure bandwidth usage, run two clients on Chrome and visit chrome://webrtc-internals


Stay in Touch
-------------

- [Follow Hayden on Twitter](https://twitter.com/haydenlee37)
- Hang out with the A-Frame community: [join the A-Frame Slack](https://aframevr-slack.herokuapp.com)
- Let us know if you've made something with Networked A-Frame! We'd love to see it!


Help and More Information
------------------------------

* Live demo site:
  * [http://haydenlee.io/networked-aframe](http://haydenlee.io/networked-aframe)
* Bugs and requests can be filed here:
  * [https://github.com/haydenjameslee/networked-aframe/issues](https://github.com/haydenjameslee/networked-aframe/issues)
* A-Frame:
  * [https://aframe.io/](https://aframe.io/)
* WebVR:
  * [https://webvr.info/](https://webvr.info/)
* EasyRTC WebRTC library:
  * [http://www.easyrtc.com/](http://www.easyrtc.com/)


Folder Structure
----------------

 * / (root)
   * Licenses and package information
 * /dist/
   * Packaged source code for deployment
 * /server/
   * Server code
 * /server/static/
   * Examples (basic.html & dance-party.html)
 * /src/
   * Client source code
 * /tests/
   * Unit tests


Roadmap
-------

* Master client concept
* Scene entities that can be defined in the HTML
* Positional audio

Interesting in adding features? Shoot me a message on [twitter](https://twitter.com/haydenlee37) or send a pull request.


License
-------

This program is free software and is distributed under an [MIT License](LICENSE).


Other A-Frame Networking Libraries
----------------------------------

[aframe-firebase-component](https://github.com/ngokevin/kframe/tree/master/components/firebase)

[aframe-webrtc](https://github.com/takahirox/aframe-webrtc)

