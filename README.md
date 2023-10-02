
# AzureWorkshop

I denne guiden skal du lære hvordan man bruker Azure Blob Storage og Azure Functions. 

Du skal lage en applikasjon hvor du kan laste opp bilder til en Azure Blob Storage og bruke en Blob Trigger til å få varsel når nye bilder lastes opp. 

# Før du begynner

Før du starter, må du sørge for at du har følgende krav oppfylt:


- Python 3.x installert.
- Pip installert for å administrere Python-pakker.
- En Azure-konto.
- Visual Studio Code installert.
- Azure Functions-utvidelse i VS Code.
- Azure CLI (https://learn.microsoft.com/en-us/cli/azure/install-azure-cli) 

# Opprett en Function App 

Først opprett du et nytt prosjekt i Visual Studio Code og åpner prosjektet.

Deretter klikker du på Azure-symbolet i menyen til venstre. 

Klikk på "+"-knappen ved siden av "Resource" i venstre meny og velg "Create function app in Azure".

Fyll inn følgende informasjon:

- Function App-navn: Velg et unikt navn for funksjonsappen din.
- Runtime Stack: Velg ".NET."
- Region: Velg en region som er geografisk nær deg.
Klikk på "Create" for å opprette funksjonsappen i Azure.

## Azure Funtion - Blob Trigger

Ved siden av "Workspace", klikk på "Create Function" (symbol med et oransjest lyn).

Fyll inn følgende detaljer:

-	Folder: Legg den i prosjektet du lagde i VS Code helt i starten
-	Select a language: C#
-	Select a .NET runtime: .NET 6
-	Select a template for your function: BlobTrigger
-	Provide a function name: BlobTrigger1
-	Provide a namespace: My.Function
-	Connection: MyBlobAppStorageConnection
-	Path: sample-workitems
  
Klikk på "Create".
Da vil du få opp en .cs fil med en blob-funksjon, som ser slik ut:

----Legge til default blobtrigger

Nå kan du igjen trykke på the oransje lynet ved siden av "Workspace" og deretter klikke på "Deploy to Function App". Velg deretter den nylig opprettede funksjonsappen når du blir bedt om det. Klikk på deploy. 

Besøk Azure-portalen (https://portal.azure.com/), og sørg for at ressursene du opprettet er synlige under riktig abonnement.

Gå inn på Storage-Accounten som er laget tilknyttet funksjonapplikasjonen du har opprettet (du finner den under "All Resources" fra forsiden). Under "Overview" ser du en verdi under "Resource group", som du kopierer og lagrer til senere. Deretter i venstre meny går du inn i "Access Key" og under "Connection String" så kopierer du verdien og lagrer også denne til senere. 

Navigerer så til "Containers" i venstre meny og lag en ny container kalt "samples-workitems". 

I terminalen i VS Code kjører du følgende kommando: 

```
az functionapp config appsettings set --name <ditt_funskjonsapp_navn> --resource-group <din_ressursgruppe> --settings MyBlobAppStorageConnection='<din_tilkoblingsstreng>'
```
Bytt ut .... ... ... 


# Oppsett av kode for opplasting av bilder til Azure Blob Storage

Nå skal vi sette opp kode for en web applikasjon der man kan laste opp bilder til den opprettede Azure Blob Storagen. 

Først kjører du følgende kommandoer i terminalen for å installere Flask og Azure Storage Blob SDK, og YAML.

```
pip install Flask azure-storage-blob
```
```
pip install PyYAML
```

## HTML-fil

Navigerer til VS Code-prosjektet og lag en mappe kalt `templates`. I denne mappen oppretter du en html-fil kalt `index.html`, som inneholder følgende kode:
```
<!DOCTYPE html>
<html>
<head>
    <title>Image Upload</title>
</head>
<body>
    <h1>Image Upload</h1>
    <form method="POST" enctype="multipart/form-data" action="/upload">
        <input type="file" name="image" accept="image/*" required>
        <input type="submit" value="Upload">
    </form>
</body>
</html>
```
Denne koden oppretter en enkel HTML-side som lar brukere laste opp bildefiler ved å velge en fil fra sin enhet og deretter sende filen til en server ved å trykke på "Upload"-knappen.

## LEGGE TIL HTML KODE FOR Å VISE BILDE

## YAML-fil

Du må også opprette en yaml-fil i prosjekt-mappen (ikke templates-mappen), kalt `config.yaml`. Legg til følgende kode i konfigurasjonsfilen:

```yaml
{
  "azure_storage_connectionstring": "din_azure_connection_string",
  "images_container_name": "din_container_navn"
}
```

Her må du endre `"din_azure_connection_string"` og `"din_container_navn"` til de faktiske verdiene som passer for ditt prosjekt. `"din_azure_connection_string"` er verdien du tidligere lagret fra Access Key, og `"din_container_navn"` er navnet på containeren som du opprettet i første steg (samples-workitems).

Sørg også for at du har riktig mappestruktur og filnavn for moduler og konfigurasjonsfiler.

## Python-fil

Deretter oppretter du en python fil i prosjekt-mappen kalt `app.py`. 

Her importerer du først de nødvendige biblotekene og modulene:

```
from flask import Flask, request, render_template, redirect, url_for
import os
import yaml
from azure.storage.blob import ContainerClient
```

Deretter oppretter du en Flask applikasjon:

```
app = Flask(__name__)

```
Så lager du en funksjon for å laste inn konfigurasjonsdata fra config.yaml-filen.

```
def load_config():
    dir_root = os.path.dirname(os.path.abspath(__file__))
    with open(dir_root + "/config.yaml", "r") as yamlfile:
        return yaml.load(yamlfile, Loader=yaml.FullLoader)
```

Definerer en rute som viser opplastingsgrensesnittet (index.html).
```
@app.route('/')
def index():
    return render_template('index.html')

```
Så lager du funksjonen for å laste opp bilder til Azure Blob Storage.

Først defineres en rute som håndterer bildeopplasting ved HTTP POST-forespørsler. 
Deretter lastes konfigurasjonsdataen inn, og en klient for Azure Blob Storage-beholderen opprettes. 

```
@app.route('/upload', methods=['POST'])
def upload():
    config = load_config()
    container_client =   ContainerClient.from_connection_string(config["azure_storage_connectionstring"], config["images_container_name"])
```

I upload-funksjonen legger du til et sjekk om en fil med navn 'image' er inkludert i forespørselen eller om filnavnet er tomt (ingen fil valgt). 

```
    if 'image' not in request.files:
        return redirect(request.url)

    image = request.files['image']

    if image.filename == '':
        return redirect(request.url)

```

Så setter du navnet på filen som lagres i Azure Blob Storage. 
   
```
    blob_name = image.filename
    blob_client = container_client.get_blob_client(blob_name)
```
Deretter sjekker du om bloben allerede eksisterer i containeren.

```
if not blob_client.exists():
        blob_client.upload_blob(image)
        return "Image uploaded successfully."
    else:
        return "Image already exists in the container. Skipping upload."
```

Til slutt må du starte Flask-applikasjonen.

```
if __name__ == '__main__':
    app.run(debug=True)
```

### Full app.py kode
Den endelige python koden i `app.py` vil se slik ut:

```python
from flask import Flask, request, render_template, redirect, url_for
import os
import yaml
from azure.storage.blob import ContainerClient

app = Flask(__name__)

def load_config():
    dir_root = os.path.dirname(os.path.abspath(__file__))
    with open(dir_root + "/config.yaml", "r") as yamlfile:
        return yaml.load(yamlfile, Loader=yaml.FullLoader)

@app.route('/')
def index():
    return render_template('index.html')

@app.route('/upload', methods=['POST'])
def upload():
    config = load_config()
    container_client = ContainerClient.from_connection_string(config["azure_storage_connectionstring"], config["images_container_name"])

    if 'image' not in request.files:
        return redirect(request.url)

    image = request.files['image']

    if image.filename == '':
        return redirect(request.url)

    blob_name = image.filename
    blob_client = container_client.get_blob_client(blob_name)

    if not blob_client.exists():
        blob_client.upload_blob(image)

        return "Image uploaded successfully."
    else:
        return "Image already exists in the container. Skipping upload."

if __name__ == '__main__':
    app.run(debug=True)


```

## Kjøre kode

For å kjøre python-koden må du kjøre denne kommandoen. Kontroller at du er i samme mappe som app.py-filen. Som ouput vil du få opp en url i terminalen som vil vise Flask-applikasjonen der du kan laste opp bilder.

```
 python -m flask run --no-debug
```
ENDRE HER...

## Teste Blob Trigger funksjonen

Kontroller at du har aktivert blobtrigger-funksjonen i Azure portalen. Dette gjør du ved å følge steget "Turn on your blob trigger" i guiden gitt tidligere (https://learn.microsoft.com/en-us/training/modules/execute-azure-function-with-triggers/8-create-blob-trigger).

Nå er du klar for å teste at funksjonen fungerer som den skal.
Gå inn i url-en til Flask-applikasjonen (sørg for at den er oppe og kjører), og last opp et bilde her. Når du får beskjed om at bildet er lastet opp, går du tilbake til blob funksjonen i Azure-portalen og nå vil du se at en blob trigger er utført. 

Nå kan du selv utforske litt hvordan dette fungerer ved å endre 'run.csx' filen som du ønsker. Her er noen forslag til ting du kan prøve:
- Legg til timestamp: Legg til tidspunkt for når bildene blir lastet opp. 
- Endre filtype: Endre Blob Trigger-funksjonen for å reagere på en bestemt type filer. For eksempel kan du legge til en feilmelding på bestemte filtyper som .jpg, .png eller .pdf.
