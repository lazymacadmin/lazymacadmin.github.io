---
layout: doc
---
### networkQuality script for Teams

```zsh
#!/bin/zsh

## Functions first

function webhook(){
    TEAMS_JSON="$1"

    curl -H "Content-Type: application/json" -d "${TEAMS_JSON}" "$2"

}

## General Variables Second
hostname=$(scutil --get ComputerName)
osVersion=$( sw_vers -productVersion )
osBuild=$( sw_vers -buildVersion )
modelName=$( /usr/libexec/PlistBuddy -c 'Print :0:_items:0:machine_name' /dev/stdin <<< "$(system_profiler -xml SPHardwareDataType)" )
wifi_port=$(networksetup -listallhardwareports | awk -F': ' '/Wi-Fi/ {
getline
print $2
sub(/^ */, "")
sub(/:$/, "") }' )
active_port=$(route get default | awk '/interface: / {print $2}')
ip=$(ifconfig $(route get default | awk '/interface: / {print $2}') | awk -F' ' '/inet / {print $2}')
ssid=$(/System/Library/PrivateFrameworks/Apple80211.framework/Resources/airport -I  | awk -F' SSID: '  '/ SSID: / {print $2}')
webhook="https://teams_webhook.fq.dn/stuff/places/things"
NQ="/usr/bin/networkQuality"

if [[ "$ssid" != ""  &&  "$active_port" == "$wifi_port" ]]; then
	TITLE="WiFi: $ssid"
else
	TITLE="Ethernet"
    ssid="Ethernet"
fi


# now start executing
output=$("$NQ" -s)

upload=$(echo "$output" | awk -F: '/Uplink capacity/ {print $2}')
download=$(echo "$output"  | awk -F: '/Downlink capacity/ {print $2}')
upresp=$(echo "$output"  | awk -F: '/Uplink Resp/ {print $2}' | awk -F'|' '{print $1}')
downresp=$(echo "$output"  | awk -F: '/Downlink Resp/ {print $2}' | awk -F'|' '{print $1}')
latency=$(echo "$output"  | awk -F: '/Latency/ {print $2}' | awk -F'|' '{print $1}')

teams_json=$( echo "{
	\"type\": \"message\",
	\"attachments\": [{
		\"contentType\": \"application/vnd.microsoft.teams.card.o365connector\",
		\"content\": {
			\"type\": \"MessageCard\",
			\"$schema\": \"https://schema.org/extensions\",
			\"summary\": \"Speed Test Results\",
			\"themeColor\": \"778eb1\",
			\"title\": \"Speed Test Results\",
			\"sections\": [{
				\"facts\": [{
					\"name\": \"Hostname\",
					\"value\": \"$hostname\"
				},{
					\"name\": \"IP Address\",
					\"value\": \"$ip\"
				},{
					\"name\": \"Model\",
					\"value\": \"$modelName\"
				},{
					\"name\": \"OS Version\",
					\"value\": \"$osVersion\"
				},{
					\"name\": \"OS Build\",
					\"value\": \"$osBuild\"
				}, {
					\"name\": \"Connection\",
					\"value\": \"$ssid\"
				}, {
					\"name\": \"Upload Speed\",
					\"value\": \"$upload\"
				}, {
					\"name\": \"Download Speed\",
					\"value\": \"$download\"
				}, {
					\"name\": \"Upload Responsiveness\",
					\"value\": \"$upresp\"
				}, {
					\"name\": \"Download Responsiveness\",
					\"value\": \"$downresp\"
				}, {
					\"name\": \"Latency\",
					\"value\": \"$latency\"
				} ]
			}]
		}
	}]
}" )

webhook "$teams_json" "$webhook"
```