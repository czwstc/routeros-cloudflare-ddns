# routeros-cloudflare-ddns

```
:local WANInterface "pppoe_out_1"
:local tempFile "ddns_wan_ip"

:local CFDomain ""
:local CFtkn ""
:local CFzoneid ""
:local CFid ""

:local CFrecordType "A"
:local CFrecordTTL "120"

:local previousIP ""
:local WANip ""

:local CFDebug "false"
:local CFCloud "false"

################# Build CF API Url (v4) #################
:local CFurl "https://api.cloudflare.com/client/v4/zones/"
:set CFurl ($CFurl . "$CFzoneid/dns_records/$CFid");

################# Get or set previous IP-variables #################
:if ($CFCloud = "true") do={
    :set WANip [/ip cloud get public-address]
} else={
    :local currentIP [/ip address get [/ip address find interface=$WANInterface ] address];
    :set WANip [:pick $currentIP 0 [:find $currentIP "/"]];
}

:if ([/file find name=$tempFile] = "") do={
    :log error "No previous ip address file found, creating..."
    :set previousIP $WANip;
    :execute script=":put $WANip" file=$tempFile;
    :log info ("CF: Updating CF, setting $CFDomain = $WANip")
    /tool fetch http-method=put mode=https output=none url="$CFurl" http-header-field="Authorization:Bearer $CFtkn,content-type:application/json" http-data="{\"type\":\"$CFrecordType\",\"name\":\"$CFDomain\",\"ttl\":$CFrecordTTL,\"content\":\"$WANip\"}"
    :error message="No previous ip address file found."
} else={
    :if ( [/file get [/file find name=$tempFile] size] > 0 ) do={ 
        :local content [/file get [/file find name=$tempFile] contents];
        :local contentLen [:len $content];  
        :local lineEnd 0;
        :local line "";
        :local lastEnd 0;   
        :set lineEnd [:find $content "\n" $lastEnd];
        :set line [:pick $content $lastEnd $lineEnd];
        :set lastEnd ($lineEnd + 1);   
        :if ([:pick $line 0 1] != "#") do={   
            :set previousIP [:pick $line 0 $lineEnd];
            :set previousIP [:pick $previousIP 0 [:find $previousIP "\r"]];
        }
    }
}

######## Write debug info to log #################
:if ($CFDebug = "true") do={
    :log info "Updating IP $CFDomain ..."
    :log info ("CF: hostname = $CFDomain")
    :log info ("CF: previousIP = $previousIP")
    :log info ("CF: currentIP = $currentIP")
    :log info ("CF: WANip = $WANip")
    :log info ("CF: CFurl = $CFurl&content=$WANip")
    :log info ("CF: Command = \"/tool fetch http-method=put mode=https url=\"$CFurl\" http-header-field=\"Authorization:Bearer $CFtkn,content-type:application/json\" output=none http-data=\"{\"type\":\"$CFrecordType\",\"name\":\"$CFDomain\",\"ttl\":$CFrecordTTL,\"content\":\"$WANip\"}\"")
};

######## Compare and update CF if necessary #####
:if ($previousIP != $WANip) do={
    :log info ("CF: Updating CF, setting $CFDomain = $WANip")
    /tool fetch http-method=put mode=https url="$CFurl" http-header-field="Authorization:Bearer $CFtkn,content-type:application/json" output=none http-data="{\"type\":\"$CFrecordType\",\"name\":\"$CFDomain\",\"ttl\":$CFrecordTTL,\"content\":\"$WANip\"}"
    /ip dns cache flush
    :if ( [/file get [/file find name=$tempFile] size] > 0 ) do={
        /file remove $tempFile
        :execute script=":put $WANip" file=$tempFile
    }
} 

```