# instructions

## decode apk

```bash

apktool d original.apk -o original_dec

jadx-gui -d original_jadx original.apk

```

> open the smali file from the original_dec in text/code editor and edit it


## recostruct apk

```bash

apktool b original_dec -o patched.apk

```

> sometimes we will need to align the zip to match the specification of android (Targetting R+: version 30 and above)

```bash

zipalign -p -f 4 ./patched.apk ./aligned_patched.apk

```

## sign apk

> create a debug keystore

```bash

keytool -genkeypair -alias labkey -keyalg RSA -keysize 2048 -validity 10000 -keystore debug.jks -storepass android -dname "CN=Lab, O=You"

```

> sign the apk

```bash

apksigner sign --ks debug.jks --ks-pass pass:android --key-pass pass:android --out patched_signed.apk patched.apk

```

## common instructions in jadx-gui

1. click on the symbol + x - find the references to the symbol
2. click on the symbol + n - change the name
3. Esc - back
4. double click


