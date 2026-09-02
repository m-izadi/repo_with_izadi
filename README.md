SSL



openssl pkcs12 -in kavosh2027.pfx -nocerts -nodes -out kvsh2027.key -legacy
openssl pkcs12 -in kavosh2027.pfx -clcerts -nokeys -out server2027.crt  -legacy
openssl pkcs12 -in kavosh2027.pfx -cacerts -nokeys -out ca-chain2027.crt    -legacy
cat server2027.crt ca-chain2027.crt > kvsh2027.crt

# تاریخ انقضا و دامنه
openssl x509 -in kvsh2027.crt -noout -subject -dates -ext subjectAltName    -legacy
# cert و key جفت باشند (هر دو hash باید یکی باشد)
openssl x509 -noout -modulus -in kvsh2027.crt | openssl md5
openssl rsa  -noout -modulus -in kvsh2027.key | openssl md5

# بکاپ سرتیفیکیت فعلی
sudo cp /data/nexus/certs/kvsh.crt /data/nexus/certs/kvsh2026.crt.bak
sudo cp /data/nexus/certs/kvsh.key /data/nexus/certs/kvsh.key2026.bak

scp -P22 kvsh2027.crt ubuntu@repo-nexus.kavosh.org:~/cert-kvsh2027/
scp -P22 kvsh2027.key ubuntu@repo-nexus.kavosh.org:~/cert-kvsh2027/

sudo cp kvsh.crt kvsh.key /data/nexus/certs/
sudo chmod 644 /data/nexus/certs/kvsh.crt
sudo chmod 600 /data/nexus/certs/kvsh.key

curl --proxy proxy-gr-p2.kavosh.org:27 -fsSL -o certum-kvsh2027.cer \ 
  http://certumdvtlsg2r39ca.repository.certum.pl/certumdvtlsg2r39ca.cer

openssl x509 -in /tmp/certum-kvsh2027.cer -inform DER \
  -out certum-kvsh2027.pem

openssl x509 -in certum-kvsh2027.pem -noout -subject -issuer

sudo cp /data/nexus/certs/kvsh2027.crt /data/nexus/certs/kvsh2027.crt.leaf-only.bak
sudo cp kvsh2027.crt kvsh2027.crt.leaf-only.bak


cat kvsh2027.crt.leaf-only.bak certum-kvsh2027.pem \
  | sudo tee kvsh2027.crt >/dev/null


8gY3NTPW
