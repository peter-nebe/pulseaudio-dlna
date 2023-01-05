# Hack for Panasonic Blu-ray Disc Player DMP-BDT500
This is a quick and dirty hack to stream MP3 audio to the DMP-BDT500.

### 1st Problem
The player's base URL sends the following response:
```
<?xml version="1.0"?>
<root xmlns="urn:schemas-upnp-org:device-1-0" xmlns:dlna="urn:schemas-dlna-org:device-1-0" xmlns:vli="urn:schemas-panasonic-com:vli">
	<specVersion>
		<major>1</major>
		<minor>0</minor>
	</specVersion>
	<device>
		<deviceType>urn:schemas-upnp-org:device:MediaRenderer:1</deviceType>
		<friendlyName>DMP-BDT500</friendlyName>
		<manufacturer>Panasonic</manufacturer>
		<modelName>Blu-ray Disc Player</modelName>
		<modelNumber>DMP-BDT500</modelNumber>
		<modelDescription>Panasonic Blu-ray Disc Player</modelDescription>
		<serialNumber></serialNumber>
		<modelURL></modelURL>
		<manufacturerURL></manufacturerURL>
		<UDN>uuid:4D454930-0100-1000-8000-CC7EE7BB07FB</UDN>
		<dlna:X_DLNADOC>DMR-1.50</dlna:X_DLNADOC>
		<serviceList>
			<service>
				<serviceType>urn:schemas-upnp-org:service:RenderingControl:1</serviceType>
				<serviceId>urn:upnp-org:serviceId:RenderingControl</serviceId>
				<SCPDURL>http://192.168.10.102:60606/Server0/RCS_SCPD</SCPDURL>
				<controlURL>http://192.168.10.102:60606/Server0/RCS_control</controlURL>
				<eventSubURL>http://192.168.10.102:60606/Server0/RCS_event</eventSubURL>
			</service>
			<service>
				<serviceType>urn:schemas-upnp-org:service:ConnectionManager:1</serviceType>
				<serviceId>urn:upnp-org:serviceId:ConnectionManager</serviceId>
				<SCPDURL>http://192.168.10.102:60606/Server0/CMS_SCPD</SCPDURL>
				<controlURL>http://192.168.10.102:60606/Server0/CMS_control</controlURL>
				<eventSubURL>http://192.168.10.102:60606/Server0/CMS_event</eventSubURL>
			</service>
			<service>
				<serviceType>urn:schemas-upnp-org:service:AVTransport:1</serviceType>
				<serviceId>urn:upnp-org:serviceId:AVTransport</serviceId>
				<SCPDURL>http://192.168.10.102:60606/Server0/AVT_SCPD</SCPDURL>
				<controlURL>http://192.168.10.102:60606/Server0/AVT_control</controlURL>
				<eventSubURL>http://192.168.10.102:60606/Server0/AVT_event</eventSubURL>
			</service>
		</serviceList>
	</device>
</root>
```
As you can see, the service URLs are absolute and start with *http*. Therefore the function *_ensure_absolute_url* does not work and the player returns the following errors:
```
"DMP-BDT500 (DLNA)" : The action "GetProtocolInfo" is not supported!
The device "DMP-BDT500 (DLNA)" failed to play! (500) - "DMP-BDT500 (DLNA)" : Could not find a suitable encoder!
```
or, with the *--disable-mimetype-check* option:
```
The device "DMP-BDT500 (DLNA)" failed to play! (500) - "DMP-BDT500 (DLNA)" : The action "SetAVTransportURI" is not supported!
```
The [hack](pulseaudio_dlna/plugins/dlna/pyupnpv2/__init__.py#L241) is to use the URLs directly.

### 2nd Problem
If the first problem is solved and *GetProtocolInfo* works, it returns the following info for MP3:
```
http-get:*:audio/mpeg:DLNA.ORG_PN=MP3
```
The last part, *DLNA.ORG_PN=MP3*, is important. If it is missing in the metadata of the MP3 stream to be played, the player returns the following error:
```
The device "DMP-BDT500 (DLNA)" failed to play! (500) - "DMP-BDT500 (DLNA)" : The command "SetAVTransportURI" failed with status code 500!
```
In debug mode you can see the player's response:
```
<s:Envelope xmlns:s="http://schemas.xmlsoap.org/soap/envelope/" s:encodingStyle="http://schemas.xmlsoap.org/soap/encoding/">
	<s:Body>
		<s:Fault>
			<faultcode>s:Client</faultcode>
			<faultstring>UPnPError</faultstring>
			<detail>
				<UPnPError xmlns="urn:schemas-upnp-org:control-1-0">
					<errorCode>714</errorCode>
					<errorDescription>Illegal MIME-type</errorDescription>
				</UPnPError>
			</detail>
		</s:Fault>
	</s:Body>
</s:Envelope>
```
The *--disable-mimetype-check* option does not help in this case. It is also not enough to correctly include the part *DLNA.ORG_PN=MP3* in the metadata that pulseaudio-dlna sends.

Luckily I found another tool that allowed me to stream MP3 audio to this player: GUPnP AV Control Point. Viewing the traffic with Wireshark showed me that the GUPnP tool sends different metadata to the player. This led to the [2nd hack](pulseaudio_dlna/plugins/dlna/pyupnpv2/__init__.py#L440): pulseaudio-dlna only sends the absolutely necessary metadata to the player. That works, but unfortunately only for the DMP-BDT500 and only with MP3.
