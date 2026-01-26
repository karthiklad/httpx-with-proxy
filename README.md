# Proxies mit HTTPX verwenden

[![Promo](https://github.com/luminati-io/Rotating-Residential-Proxies/blob/main/50%25%20off%20promo.png)](https://brightdata.de/proxy-types/residential-proxies) 

Dieser Leitfaden erklärt, wie Sie Proxies mit HTTPX verwenden, mit Beispielen für nicht authentifizierte, authentifizierte, rotierende und Fallback-Proxies.

## Nicht authentifizierte Proxies verwenden

Bei einem nicht authentifizierten Proxy verwenden wir keinen Benutzernamen oder kein Passwort, und alle Anfragen gehen an eine `proxy_url`. Unten finden Sie ein Code-Snippet, das einen nicht authentifizierten Proxy verwendet:

```python
import httpx

proxy_url = "http://localhost:8030"


with httpx.Client(proxy=proxy_url) as client:
    ip_info = client.get("https://geo.brdtest.com/mygeo.json")
    print(ip_info.text)
```

## Authentifizierte Proxies verwenden

Authentifizierte Proxies erfordern einen Benutzernamen und ein Passwort. Sobald Sie korrekte Zugangsdaten übermitteln, erhalten Sie Zugriff auf die Verbindung zum Proxy.

Mit Authentifizierung sieht die `proxy_url` so aus: `http://<username>:<password>@<proxy_url>:<port_number>`. Das folgende Beispiel zeigt, wie Sie den Benutzerteil der Authentifizierungszeichenfolge mithilfe von `zone` und `username` konstruieren. Außerdem werden [Bright Data's datacenter proxies](https://brightdata.de/proxy-types/datacenter-proxies) als Basisverbindung verwendet.

```python
import httpx

username = "your-username"
zone = "your-zone-name"
password = "your-password"

proxy_url = "http://brd-customer-{username}-zone-{zone}:{password}@brd.superproxy.io:33335"

ip_info = httpx.get("https://geo.brdtest.com/mygeo.json", proxy=proxy_url)

print(ip_info.text)
```

Hier ist die Aufschlüsselung des obigen Codes:

- Wir beginnen mit dem Erstellen von Konfigurationsvariablen: `username`, `zone` und `password`.
- Wir verwenden diese, um unsere `proxy_url` zu erstellen: `"http://brd-customer-{username}-zone-{zone}:{password}@brd.superproxy.io:33335"`.
- Wir senden eine Anfrage an die API, um allgemeine Informationen über unsere Proxy-Verbindung abzurufen.

Die Antwort sollte so aussehen.

```json
{"country":"US","asn":{"asnum":20473,"org_name":"AS-VULTR"},"geo":{"city":"","region":"","region_name":"","postal_code":"","latitude":37.751,"longitude":-97.822,"tz":"America/Chicago"}}

```

## Rotierende Proxies verwenden

Rotierende Proxies bedeuten, eine Liste von Proxies zu erstellen und zufällig daraus auszuwählen. Das folgende Code-Snippet erstellt eine Liste von `countries` und verwendet dann bei jeder Anfrage `random.choice()`, um ein zufälliges Land aus der Liste auszuwählen. Unsere `proxy_url` wird entsprechend formatiert, um zum Land zu passen. Die Liste der [rotating proxies](https://brightdata.de/solutions/rotating-proxies) im Code stammt von Bright Data.

```python
import httpx
import asyncio
import random


countries = ["us", "gb", "au", "ca"]
username = "your-username"
proxy_url = "brd.superproxy.io:33335"

datacenter_zone = "your-zone"
datacenter_pass = "your-password"


for random_proxy in countries:
    print("----------connection info-------------")
    datacenter_proxy = f"http://brd-customer-{username}-zone-{datacenter_zone}-country-{random.choice(countries)}:{datacenter_pass}@{proxy_url}"

    ip_info = httpx.get("https://geo.brdtest.com/mygeo.json", proxy=datacenter_proxy)

    print(ip_info.text)
```

Der Code ist dem für authentifizierte Proxies sehr ähnlich. Die wichtigsten Unterschiede sind:

- Wir erstellen ein Array von Ländern: `["us", "gb", "au", "ca"]`.
- Wir führen mehrere Anfragen statt nur einer aus. Für jede Anfrage verwenden wir `random.choice(countries)`, um bei jeder Erstellung der `proxy_url` jedes Mal ein zufälliges Land auszuwählen.

## Eine Fallback-Proxy-Verbindung erstellen

Alle obigen Beispiele verwenden Rechenzentrums- und freie Proxies. Erstere werden von Websites oft blockiert, letztere sind nicht sehr zuverlässig. Damit das alles funktioniert, sollte es einen Fallback auf Residential Proxies geben.

Erstellen wir dazu eine Funktion namens `safe_get()`. Wenn wir sie aufrufen, versuchen wir zuerst, die URL über eine Rechenzentrumsverbindung abzurufen. Wenn dies fehlschlägt, _fallen wir zurück_ auf unsere Residential-Verbindung.

```python
import httpx
from bs4 import BeautifulSoup
import asyncio


country = "us"
username = "your-username"
proxy_url = "brd.superproxy.io:33335"

datacenter_zone = "datacenter_proxy1"
datacenter_pass = "datacenter-password"

residential_zone = "residential_proxy1"
residential_pass = "residential-password"

cert_path = "/home/path/to/brightdata_proxy_ca/New SSL certifcate - MUST BE USED WITH PORT 33335/BrightData SSL certificate (port 33335).crt"


datacenter_proxy = f"http://brd-customer-{username}-zone-{datacenter_zone}-country-{country}:{datacenter_pass}@{proxy_url}"

residential_proxy = f"http://brd-customer-{username}-zone-{residential_zone}-country-{country}:{residential_pass}@{proxy_url}"

async def safe_get(url: str):
    async with httpx.AsyncClient(proxy=datacenter_proxy) as client:
        print("trying with datacenter")
        response = await client.get(url)
        if response.status_code == 200:
            soup = BeautifulSoup(response.text, "html.parser")
            if not soup.select_one("form[action='/errors/validateCaptcha']"):
                print("response successful")
                return response
    print("response failed")
    async with httpx.AsyncClient(proxy=residential_proxy, verify=cert_path) as client:
        print("trying with residential")
        response = await client.get(url)
        print("response successful")
        return response

async def main():
    url = "https://www.amazon.com"
    response = await safe_get(url)
    with open("out.html", "w") as file:
        file.write(response.text)

asyncio.run(main())
```

Hier ist die Aufschlüsselung des Codes:

- Wir haben nun zwei Sätze von Konfigurationsvariablen: einen für unsere Rechenzentrumsverbindung und einen weiteren für unsere Residential-Verbindung.
- Dieses Mal verwenden wir eine `AsyncClient()`-Sitzung, um einige der fortgeschritteneren Funktionen von HTTPX einzuführen.
- Zuerst versuchen wir, unsere Anfrage mit dem `datacenter_proxy` zu stellen.
- Wenn es uns nicht gelingt, eine geeignete Antwort zu erhalten, wiederholen wir die Anfrage mit dem `residential_proxy`. Beachten Sie auch das `verify`-Flag im Code. Bei der Verwendung von Bright Data's Residential Proxies müssen Sie das [SSL certificate](https://docs.brightdata.com/general/account/ssl-certificate) herunterladen und verwenden.
- Sobald wir eine solide Antwort erhalten haben, schreiben wir die Seite in eine HTML-Datei. Wir können diese Seite in einem Browser öffnen und sehen, worauf der Proxy tatsächlich zugegriffen hat und was an uns zurückgesendet wurde.

Nach dem Ausführen des obigen Codes sollten Ihre Ausgabe und die resultierende HTML-Datei so aussehen.

```
trying with datacenter
response failed
trying with residential
response successful
```

![Screenshot of the Amazon homepage](https://github.com/luminati-io/httpx-with-proxy/blob/main/Images/image.png)

## Fazit

Wenn Sie HTTPX mit [Bright Data's top-tier proxy services](https://brightdata.de/proxy-types) kombinieren, erhalten Sie eine private, effiziente und zuverlässige Möglichkeit, das Web zu Scraping. Starten Sie noch heute Ihre kostenlose Testversion mit den Proxies von Bright Data!