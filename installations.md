# setting up 

snap install android-studio --classic

install:
1. burpsuite
2. gnirehtet
3. wireshark

## adding ca to the device's system CA's

before installing the CA we need to convert it to PEM format and rename it using the following:

> convert your certificate to PEM format, for example from DER format

```bash 
    openssl x509 -in <DER CERT> -out <PEM CERT>
```

> obtain the hash of your certificate

```bash
    openssl x509 -inform PEM -subject_hash_old -in <PEM CERT> | head -1
```

> copy the certificate to a new file with the "hash_name.0"

```bash
    cp <PEM CERT> <HASH VALUE>.0
```


now we need to install the CA certificate, here are some ways of adding it:

1. bind method:

```bash
    cp -r /system/etc/security/cacerts /data/local/tmp/cacerts
    mount -o bind /data/local/tmp/cacerts /system/etc/security/cacerts

    adb push <CERT> /data/local/tmp/cacerts/
    adb shell chmod 664 /data/local/tmp/cacerts/<CERT>
```

2. remounting - requires adbd to run as root (special root)

```bash
    adb root
    adb remount 
    adb push <CERT> /system/etc/security/cacerts/
    adb shell chmod 644 /system/etc/security/cacerts/<CERT>
    adb reboot
```


3. android 14 - special change to location of root CAs
    a. creating the cacerts directory

```bash
        mount -t tmpfs tmpfs /system/etc/security/cacerts
        cp /apex/com.android.conscrypt/cacerts/* /system/etc/security/cacerts/

        cp <burp certificates> /system/etc/security/cacerts/

        chown root:root /system/etc/security/cacerts/

        chown root:root /system/etc/security/cacerts/*
        chmod 644 /system/etc/security/cacerts/*

        chcon u:subject_r:system_file:s0 /system/etc/security/cacerts/*
```

    b. updating all processes namespaces

```bash
    nsenter --mount=/proc/<PID>/ns/mnt -- /bin/mount --bind /system/etc/security/cacerts /apex/com.android.conscrypt/cacerts

    # PID is the pid of the zygote64 process
```


## adding ca to the device's system CA's - without root

1. unpack the apk:

```bash
    apktool d <APK FILE>
```

2. open the AndroidManifest.xml and look for the application tag.
3. you should see something like the following:

```xml
    <application ... android:name="..." android:NetworkSecurityConfig="@xml/{network security config filename}" ...>
```

4. edit the network security config file specified in the networkSecurityConfig attribute (for example @xml/network_security_config.xml would be in **./res/xml/network_security_config.xml**)
to add a custom CA, the config should look like:

```xml

<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <base-config {... attributes}>
        <trust-anchors>
            <certificates src="@raw/custom_ca" />
            <certificates src="system" />
        </trust-anchors>
    </base-config>
</network-security-config>

```

5. where the "trust anchors" tag can contain multiple "certificates" tags with different sources (in the example above, src="system" refers to CAs trusted by the system).

6. then, you can just add the CA certificate in PEM or DER format to the location you specified in the "src" attribute (in the example above, it would be ./res/raw/custom_ca)

7. finally, rebuild the apk, zipalign and resign it.

imo 2024.05.1091

zello