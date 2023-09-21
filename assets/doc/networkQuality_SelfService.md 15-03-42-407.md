---
layout: null
---
## 

### networkQuality script for Self Service with Summary Upload

```zsh
#!/bin/zsh

## Functions first
function dialog_command(){
	echo "$1"  >> "${commandFile}"
}

function summary_command(){
	echo "$1"  >> "${summary_report}"
}

function prepare_summary(){
    download_speed=$(cat "${outputFileName}" | plutil -extract dl_throughput raw - )
    upload_speed=$(cat "${outputFileName}" | plutil -extract ul_throughput raw - )
    download=$(( download_speed / 1000000 ))
    upload=$(( upload_speed / 1000000 ))
    interface=$(cat "${outputFileName}" | plutil -extract interface_name raw - )
    if [[ "$ssid" != ""  &&  "$active_port" == "$wifi_port" ]]; then
   	conn_info=$(/System/Library/PrivateFrameworks/Apple80211.framework/Versions/Current/Resources/airport -I)
    else
        conn_info=$(ifconfig "$interface")
    fi
    summary_command "networkQuality Summary for $hostname\n"
    summary_command "$hostname is a $modelName running macOS $osVersion, Build $osBuild, and connected via $ssid on $interface"
    summary_command "Connection info:"
    summary_command "$conn_info"
    summary_command "$loggedInUser is logged in"
    summary_command "Download speed is $download Mbps,"
    summary_command "Upload speed is $upload  Mbps."
}

function upload_summary(){
    ## Get the JSS URL from the Mac the script is running on
    jssURL=$(defaults read /Library/Preferences/com.jamfsoftware.jamf.plist jss_url | sed 's|/$||')

    ### API role client_id and client_secret
    client_id="jamf_client_id" # Switch to $4 if using script options in a policy
    client_secret="jamf_client_secret" # Switch to $5 if using script options in a policy
    ### You can also pass the API credentials as script parameters in the policy ($4 and $5) instead of hardcoding them in
    #client_id="$4"
    #client_secret="$5"

    ##    Get a bearer token from api user
    output=$(curl --location --request POST "${jssURL}/api/oauth/token" \
        --header "Content-Type: application/x-www-form-urlencoded" \
        --data-urlencode "client_id=${client_id}" \
        --data-urlencode "grant_type=client_credentials" \
        --data-urlencode "client_secret=${client_secret}"
    )

    auth_token=$(echo "$output" | plutil -extract access_token raw -)
    auth_token_expires=$(echo "$output" | plutil -extract expires_in raw -)
    ## Pull ioreg data on the Mac
    ioregData=$(ioreg -rd1 -c IOPlatformExpertDevice)

    ## Get the Mac's Serial Number and UUID
    SerialNum=$(awk -F'"' '/IOPlatformSerialNumber/{print $4}' <<< "$ioregData")
    UUID=$(awk -F'"' '/IOPlatformUUID/{print $4}' <<< "$ioregData")

    ## Determine which one to use to pull the Mac's JSS ID
    if [[ -z "$SerialNum" ]] && [[ ! -z "$UUID" ]]; then
        sourceID="$UUID"
        sourceString="uuid"
    elif [[ ! -z "$SerialNum" ]]; then
        sourceID="$SerialNum"
        sourceString="serialnumber"
    fi

    ## Get the Mac's JSS ID via API call
    jssID=$(curl -H " : text/xml" -H "Authorization: Bearer $auth_token" -sfk "${jssURL}/JSSResource/computers/${sourceString}/${sourceID}/subset/general" | xmllint --format - | awk -F'>|<' '/<id>/{print $3; exit}')

    # upload now
    curl -X POST "${jssURL}/api/v1/computers-inventory/$jssID/attachments" -H "accept: application/json" -H "Authorization: Bearer $auth_token" -H "Content-Type: multipart/form-data" -F "file=@${summary_report};type=text/plain"

}

function finalize(){
    prepare_summary
	dialog_command "progresstext: Finishing Speed Test Calculations"
	dialog_command "progress: 85"
    dialog_command "message: Download speed: $download Mbps<br>Upload speed: $upload Mbps"
    sleep 2
    dialog_command "message: Download speed: $download Mbps<br>Upload speed: $upload Mbps"
    sleep 2
    dialog_command "progresstext: Uploading results"
	dialog_command "progress: 91"
    upload_summary
    sleep 4
	dialog_command "progresstext: Testing Complete"
	dialog_command "progress: complete"
	dialog_command "button1text: Done"
	dialog_command "button1: enable"
    rm -f ${commandFile}
    exit 0
}

## General Variables Second
hostname=$(scutil --get ComputerName)
loggedInUser=$( scutil <<< "show State:/Users/ConsoleUser" | awk -F': ' '/[[:space:]]+Name[[:space:]]:/ { if ( $2 != "loginwindow" ) { print $2 }}' )
osVersion=$( sw_vers -productVersion )
osBuild=$( sw_vers -buildVersion )
modelName=$( /usr/libexec/PlistBuddy -c 'Print :0:_items:0:machine_name' /dev/stdin <<< "$(system_profiler -xml SPHardwareDataType)" )
wifi_port=$(networksetup -listallhardwareports | awk -F': ' '/Wi-Fi/ {
getline
print $2
sub(/^ */, "")
sub(/:$/, "") }' )
active_port=$(route get default | awk '/interface: / {print $2}')
ssid=$(/System/Library/PrivateFrameworks/Apple80211.framework/Resources/airport -I  | awk -F' SSID: '  '/ SSID: / {print $2}')

## Script-specific variables
outputFileName="/var/tmp/networkQualityTest.log"
summary_report="/var/tmp/networkQuality_report_$hostname.log"
DIALOG_APP="/Library/Application Support/Dialog/Dialog.app"
NQ="/usr/bin/networkQuality"
DIALOG_PATH="/usr/local/bin/dialog"
commandFile="/var/tmp/dialog.log"
message="Starting Internet Speed Test"
ICON="SF=gauge.high"  # SF Symbols 4

if [[ -f "${outputFileName}" ]]; then
    rm -f "${outputFileName}"
fi
if [[ -f "${summary_report}" ]]; then
    rm -rf "${summary_report}"
fi

if [[ "$ssid" != ""  &&  "$active_port" == "$wifi_port" ]]; then
	TITLE="WiFi: $ssid"
else
	TITLE="Ethernet"
    ssid="Ethernet"
fi

open -a "${DIALOG_APP}" --args --title "$TITLE - Speed Test" --icon "${ICON}" --small --progress 100 --message "Please wait..." --commandfile "$commandFile" --progresstext " " --button1text "Please Wait" --button1disabled
dialogCMD="${DIALOG_PATH} --title \"Speed Test\" \
--message \"$message\" \
--alignment center \
--icon \"$ICON\" \
--centericon \
--commandfile \"$commandFile\" \
--ontop \
--moveable \
--small \
--progress 3 \
--button1text \"Please Wait\" \
--button1disabled"

# Launch dialog and run it in the background sleep for a second to let thing initialise
$dialogCMD &
sleep 2

# now start executing

dialog_command "progress: 0"
dialog_command "progresstext: Starting..."

# download the installer package and capture the % percentage sign progress for Dialog display
"$NQ" -s -c > "${outputFileName}" &
NQ_PID=$!
echo "PID is $NQ_PID"
running=true
i=0

while $running; do
    sleep 1
    kill -0 $NQ_PID 2>/dev/null

    if [[ $? -eq 0 ]]; then
        if (( i < 5 )); then
            dialog_command "progresstext: Testing Download Speed..."
            dialog_command "progress: 33"
        else
            dialog_command "progresstext: Testing Upload Speed..."
            dialog_command "progress: 66"
        fi
        ((i++))
        sleep 2
    else
        running=false
    fi
done

finalize

```
