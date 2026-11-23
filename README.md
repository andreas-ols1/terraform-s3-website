# Terraform med AWS S3 og statiske websider

## Oppgaven

I denne øvelsen skal du hoste en React-applikasjon som viser kryptovaluta-informasjon. Applikasjonen er allerede bygget og klar til å deployes. Du skal lære hvordan Terraform brukes til å sette opp infrastrukturen som hoster den på AWS.

Du vil bruke **Infrastructure as Code (IaC)** for å automatisere hele prosessen med å sette opp:
- En S3 bucket for å hoste nettsiden
- CloudFront CDN for global distribusjon og HTTPS
- DNS-konfigurasjon for custom domenenavn
- CI/CD pipeline for automatisk deployment

## Du vil lære

Gjennom denne øvelsen lærer du om:

- **Terraform grunnleggende**: Ressurser, variabler, outputs og state management
- **AWS S3 Website Hosting**: Konfigurasjon av S3 buckets for statiske nettsider
- **CloudFront CDN**: Global distribusjon med HTTPS og caching
- **Terraform-moduler**: Bygge gjenbrukbar infrastruktur-kode
- **Remote State**: Håndtere Terraform state i team-miljøer
- **CI/CD med GitHub Actions**: Automatisere infrastruktur-deployment
- **Infrastructure as Code**: Best practices for å versjonere og administrere infrastruktur

## AWS-tjenester i denne labben

- **S3 (Simple Storage Service)**: Objektlager. Brukes til å hoste de statiske filene (HTML, CSS, JS) som utgjør nettsiden, og til å lagre Terraform state remote.
- **CloudFront**: AWS sitt CDN. Distribuerer nettsiden globalt og legger HTTPS på toppen av S3.
- **Route53**: AWS sin DNS-tjeneste. Brukes til å peke et custom domenenavn mot CloudFront-distribusjonen.
- **ACM (Certificate Manager)**: Utsteder og håndterer TLS-sertifikater. Brukes til å gi CloudFront et gyldig HTTPS-sertifikat for custom domenet.
- **IAM**: Identity and Access Management. Brukes gjennom bucket policies og GitHub Actions-credentials for å styre hvem som kan lese og endre hva.

## Forberedelser

### Om GitHub forks

En **fork** er din egen kopi av et GitHub-repo under din egen konto. Du jobber i din kopi uten å påvirke originalen, og kan senere åpne pull requests tilbake hvis du vil bidra endringer. I denne labben trenger du en fork av to grunner: du må kunne pushe commits for å teste CI/CD-pipelinen, og GitHub Actions-workflowen kjører mot secrets du selv legger inn i ditt eget repo.

### Steg 0: Opprett GitHub Codespace fra din fork

1. **Fork dette repositoriet** til din egen GitHub-konto
2. **Åpne Codespace**: Klikk på "Code" → "Codespaces" → "Create codespace on main"
3. **Vent på at Codespace starter**: Dette kan ta et par minutter første gang
4. **Terminalvindu**: Du vil utføre de fleste kommandoer i terminalen som åpner seg nederst i Codespace

**Tips**: Trykk `.` (punktum) når du er i et GitHub repository for å åpne det direkte i en nettleser-basert VS Code editor. Dette er raskere enn å starte en full Codespace for små editeringer.

Konfigurer AWS-nøkler i Codespace
Terraform og AWS CLI trenger AWS-nøkler for å kunne snakke med AWS-kontoen din. En Codespace starter uten disse.

Du kan få utdelt Access Key ID og Secret Access Key i klasserommet, eller lage dem selv i IAM — se https://github.com/glennbechdevops/aws-iam-accesskeys. 

### Konfigurer AWS-nøkler i Codespace

Terraform og AWS CLI trenger AWS-nøkler for å kunne snakke med AWS-kontoen din. En Codespace starter uten disse, så du må konfigurere dem én gang per Codespace.

Hent `Access Key ID` og `Secret Access Key`:

```bash
aws configure
```

Fyll inn verdiene når du blir spurt:

- **AWS Access Key ID**: fra kontoen din
- **AWS Secret Access Key**: fra kontoen din
- **Default region name**: `eu-west-1`
- **Default output format**: `json`


**Merk**: Nøklene lagres i `~/.aws/credentials` inne i Codespacen. Hvis Codespacen slettes eller resettes, må du kjøre `aws configure` på nytt.

### Steg 1: Verifiser miljøet

Repositoriet er allerede klonet i ditt Codespace. Verifiser at du er i riktig mappe:

```bash
pwd
ls
```

Du skal se filene fra dette repositoriet, inkludert mappen `s3_demo_website`. 

### Steg 2: Opprett Terraform-konfigurasjon

Nå skal du bygge opp Terraform-konfigurasjonen fra bunnen av. Du vil lære om de ulike AWS S3-ressursene som trengs for å hoste en statisk nettside.

1. **Opprett `providers.tf`** i rotmappen av prosjektet:

```hcl
terraform {
  required_version = ">= 1.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "eu-west-1"
}

# Alias provider for us-east-1
# Nødvendig for CloudFront ACM-sertifikater senere i oppgaven
provider "aws" {
  alias  = "us-east-1"
  region = "us-east-1"
}
```

**Forklaring:**
- `required_version` sikrer at Terraform-versjonen er minst 1.0
- `required_providers` spesifiserer AWS provider versjon (~> 5.0 betyr versjon 5.x)
- Default provider bruker `eu-west-1`
- Aliased provider for `us-east-1` vil brukes senere for ACM-sertifikater (se [bonusoppgaven om egendefinert domene](#bonusoppgave-egendefinert-domene-med-https-acm--cloudfront))

2. **Opprett `main.tf`** i rotmappen av prosjektet

3. **Opprett S3 bucket-ressursen** med et hardkodet bucket-navn (erstatt `<unikt-bucket-navn>` med ditt eget unike navn, f.eks. dine initialer eller studentnummer):

**Viktig om bucket-navn:**
- S3 bucket-navn må være **globalt unike** på tvers av alle AWS-kontoer i hele verden
- Hvis noen andre allerede bruker navnet "my-website", kan du ikke bruke det samme navnet
- Bruk derfor noe unikt som dine initialer, studentnummer, eller en kombinasjon: `glennbech-pgr301-website`
- Det er også strenge regler for formatet: https://docs.aws.amazon.com/AmazonS3/latest/userguide/bucketnamingrules.html

```hcl
resource "aws_s3_bucket" "website" {
  bucket = "unikt-bucket-navn"  # Bytt til noe globalt unikt
}
```

4. **Konfigurer S3 bucket for website hosting**:

```hcl
resource "aws_s3_bucket_website_configuration" "website" {
  bucket = aws_s3_bucket.website.id

  index_document {
    suffix = "index.html"
  }

  error_document {
    key = "error.html"
  }
}
```

5. **Åpne bucketen for offentlig tilgang** (nødvendig for static websites):

```hcl
resource "aws_s3_bucket_public_access_block" "website" {
  bucket = aws_s3_bucket.website.id

  block_public_acls       = false
  block_public_policy     = false
  ignore_public_acls      = false
  restrict_public_buckets = false
}
```

6. **Legg til en bucket policy som tillater offentlig lesing**:

```hcl
resource "aws_s3_bucket_policy" "website" {
  bucket = aws_s3_bucket.website.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid       = "PublicReadGetObject"
        Effect    = "Allow"
        Principal = "*"
        Action    = "s3:GetObject"
        Resource  = "${aws_s3_bucket.website.arn}/*"
      }
    ]
  })

  depends_on = [aws_s3_bucket_public_access_block.website]
}
```

7. **Legg til en output for å få URL-en til nettsiden**:

```hcl
output "s3_website_url" {
  value       = "http://${aws_s3_bucket_website_configuration.website.website_endpoint}"
  description = "URL for the S3 hosted website"
}
```

### Steg 3: Deploy infrastrukturen

Nå er du klar til å deploye infrastrukturen. Sørg for at du har erstattet `unikt-bucket-navn` i `main.tf` med ditt eget unike navn.

```bash
terraform init
terraform apply
```

Du vil nå få en 404 hvis du åpner URLen i nettleser fordi S3-Bucketen ikke har noe index.html ennå. 

### Steg 4: Bygg React-applikasjonen

Før vi kan laste opp nettsiden til S3, må vi bygge React-applikasjonen. Dette kompilerer TypeScript-koden og optimaliserer alle assets for produksjon.

1. **Naviger til applikasjonsmappen**:

```bash
cd s3_demo_website
```

2. **Installer dependencies** (hvis ikke allerede gjort):

```bash
npm install
```

3. **Bygg applikasjonen**:

```bash
npm run build
```

Dette vil opprette en `dist`-mappe med den ferdige produksjonsklare nettsiden.

4. **Gå tilbake til rotmappen**:

```bash
cd ..
```

### Steg 5: Last opp filer til S3

Nå kan vi laste opp den bygde nettsiden til S3 bucketen ved hjelp av AWS CLI:

```bash
aws s3 sync s3_demo_website/dist s3://unikt-bucket-navn
```

Legg merke til at vi synkroniserer `dist`-mappen, ikke hele `s3_demo_website`-mappen. `dist`-mappen inneholder kun de optimaliserte filene som trengs for produksjon.

### Steg 6: Inspiser bucketen i AWS Console

Gå til AWS Console, og tjenesten S3, og se på objekter og bucket-egenskaper for å forstå hvordan alt er satt opp.

### Steg 7: Åpne nettsiden

Hent URL-en til nettsiden:

```bash
terraform output s3_website_url
```

Åpne URL-en i nettleseren for å se din statiske nettside.

### Steg 8: Refaktorer til å bruke variabler

Nå som du har fått infrastrukturen til å fungere med hardkodet bucket-navn, skal vi gjøre konfigurasjonen mer fleksibel ved å introdusere variabler.

1. **Legg til en variabel for bucket-navnet** øverst i `main.tf`:

```hcl
variable "bucket_name" {
  description = "The name of the S3 bucket"
  type        = string
}
```

2. **Erstatt det hardkodede bucket-navnet** i S3 bucket-ressursen:

```hcl
resource "aws_s3_bucket" "website" {
  bucket = var.bucket_name  # Endret fra hardkodet verdi
}
```

3. **Apply endringene** med variabelen:

```bash
terraform apply -var 'bucket_name=ditt_bucket_navn'
```

Terraform vil nå vise at det ikke er nødvendig med endringer, siden bucket-navnet er det samme.

**Fordelen med variabler**: Du kan nå enkelt endre bucket-navnet uten å redigere koden, og gjenbruke samme konfigurasjon for flere miljøer.

### Steg 9: Bruk default-verdier for variabler

I stedet for å måtte oppgi verdier på kommandolinjen hver gang, kan du sette default-verdier for variabler. Dette gjør det enklere å jobbe med Terraform i daglig bruk.

1. **Oppdater variabelen med en default-verdi**:

```hcl
variable "bucket_name" {
  description = "The name of the S3 bucket"
  type        = string
  default     = "ditt-bucket-navn"  # Erstatt med ditt eget unike navn
}
```

2. **Apply uten å spesifisere variabel**:

```bash
terraform apply
```

Terraform vil nå bruke default-verdien uten at du må oppgi den på kommandolinjen.

**Best practice**: Bruk default-verdier for variabler som sjelden endres, men la kritiske verdier (som bucket-navn i produksjon) være uten default for å sikre at de blir eksplisitt satt.

### Bonusoppgave: Modifiser nettsiden

Prøv å endre HTML- og CSS-filene i `s3_demo_website`-mappen, og kjør sync-kommandoen på nytt for å se endringene:

```bash
  aws s3 sync s3_demo_website/dist s3://unikt-bucket-navn
```

### Bonusoppgave: Egendefinert domene med HTTPS (ACM + CloudFront)

Så langt serveres nettsiden over `http://` på en S3-website-URL. I denne bonusoppgaven legger du en CloudFront-distribusjon foran S3-bucketen, utsteder et TLS-sertifikat med ACM, og peker et eget domenenavn mot distribusjonen via Route53. Resultatet er en nettside som svarer på `https://ditt-domene` med et gyldig sertifikat.

Her får du endelig bruk for `us-east-1`-provideren du satte opp i `providers.tf`: CloudFront krever at ACM-sertifikatet ligger i `us-east-1`, uansett hvilken region resten av infrastrukturen din bruker.

Dette er en flerbruker-øving: alle studentene deler den samme hosted zonen `thecloudcollege.com`. For å unngå kollisjoner får hver student sitt eget **unike subdomene** (f.eks. `glennbech.thecloudcollege.com`), styrt av variabelen `student_id`.

> **Forutsetning**: Alle studentene har en IAM-bruker i den samme AWS-kontoen, og den kontoen eier hosted zonen `thecloudcollege.com`. Både ACM-valideringen og alias-recorden skrives inn i denne delte sonen, så et unikt `student_id` er det som holder studentenes records fra hverandre.

1. **Legg til en variabel for din student-ID** i `main.tf`. Sett din egen unike student-ID som `default`, slik at du kan kjøre `terraform apply` uten å oppgi den på kommandolinjen hver gang:

```hcl
variable "student_id" {
  description = "Din unike student-ID (initialer eller studentnummer). Brukes som subdomene, og må være unik blant studentene."
  type        = string
  default     = "dittnavn" # Erstatt med din egen unike student-ID
}

locals {
  # Felles hosted zone som alle studentene deler
  zone_name = "thecloudcollege.com"

  # Hver student får sitt eget subdomene, f.eks. glennbech.thecloudcollege.com
  domain_name = "${var.student_id}.${local.zone_name}"
}
```

2. **Slå opp den delte hosted zonen** (Terraform finner zone-IDen for deg, så du slipper å hardkode den). Merk at oppslaget bruker apex-domenet `thecloudcollege.com`, ikke ditt subdomene:

```hcl
data "aws_route53_zone" "main" {
  name         = local.zone_name
  private_zone = false
}
```

3. **Be ACM om et sertifikat** i `us-east-1`, med DNS-validering:

```hcl
resource "aws_acm_certificate" "website" {
  provider          = aws.us-east-1
  domain_name       = local.domain_name
  validation_method = "DNS"

  lifecycle {
    create_before_destroy = true
  }
}
```

4. **Opprett valideringsrecordene i Route53**. ACM gir deg en CNAME-record per domene som beviser at du eier domenet; `aws_acm_certificate_validation` venter til valideringen er fullført:

```hcl
resource "aws_route53_record" "cert_validation" {
  for_each = {
    for dvo in aws_acm_certificate.website.domain_validation_options : dvo.domain_name => {
      name   = dvo.resource_record_name
      record = dvo.resource_record_value
      type   = dvo.resource_record_type
    }
  }

  zone_id = data.aws_route53_zone.main.zone_id
  name    = each.value.name
  type    = each.value.type
  records = [each.value.record]
  ttl     = 60
}

resource "aws_acm_certificate_validation" "website" {
  provider                = aws.us-east-1
  certificate_arn         = aws_acm_certificate.website.arn
  validation_record_fqdns = [for record in aws_route53_record.cert_validation : record.fqdn]
}
```

5. **Opprett CloudFront-distribusjonen** med S3-website-endepunktet som origin. Siden dette er et website-endepunkt (ikke REST-endepunktet), brukes `custom_origin_config` med `http-only` mot origin, mens `viewer_protocol_policy` tvinger brukere over på HTTPS:

> **Tålmodighet**: Utrulling av en CloudFront-distribusjon tar vanligvis 5–10 minutter. `terraform apply` vil stå og vise `Still creating...` i hele denne perioden — det er helt normalt. **Ikke avbryt med Ctrl-C**, da kan Terraform-staten bli korrupt.

```hcl
resource "aws_cloudfront_distribution" "website" {
  enabled             = true
  default_root_object = "index.html"
  aliases             = [local.domain_name]

  origin {
    domain_name = aws_s3_bucket_website_configuration.website.website_endpoint
    origin_id   = "s3-website"

    custom_origin_config {
      http_port              = 80
      https_port             = 443
      origin_protocol_policy = "http-only"
      origin_ssl_protocols   = ["TLSv1.2"]
    }
  }

  default_cache_behavior {
    allowed_methods        = ["GET", "HEAD"]
    cached_methods         = ["GET", "HEAD"]
    target_origin_id       = "s3-website"
    viewer_protocol_policy = "redirect-to-https"

    forwarded_values {
      query_string = false
      cookies {
        forward = "none"
      }
    }
  }

  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }

  viewer_certificate {
    acm_certificate_arn      = aws_acm_certificate_validation.website.certificate_arn
    ssl_support_method       = "sni-only"
    minimum_protocol_version = "TLSv1.2_2021"
  }
}
```

6. **Pek domenet mot CloudFront** med en Route53 alias-record. En alias-record er en AWS-spesifikk record som peker direkte på en AWS-ressurs uten å koste et ekstra DNS-oppslag:

```hcl
resource "aws_route53_record" "website" {
  zone_id = data.aws_route53_zone.main.zone_id
  name    = local.domain_name
  type    = "A"

  alias {
    name                   = aws_cloudfront_distribution.website.domain_name
    zone_id                = aws_cloudfront_distribution.website.hosted_zone_id
    evaluate_target_health = false
  }
}
```

7. **Legg til en output for HTTPS-URL-en**:

```hcl
output "website_https_url" {
  value       = "https://${local.domain_name}"
  description = "HTTPS-URL for nettsiden via CloudFront"
}
```

8. **Apply endringene**:

```bash
terraform apply
```

Legg merke til at `terraform apply` kan bruke flere minutter: ACM-validering og utrulling av CloudFront-distribusjonen tar tid. Når det er ferdig, åpne `https://<student_id>.thecloudcollege.com` i nettleseren og bekreft at sertifikatet er gyldig (hengelås i adresselinjen).

## Appendix B: Privat bucket i en produksjonssetting

Oppsettet over lar S3-bucketen være offentlig lesbar. CloudFront fungerer da som et lag foran en bucket som fortsatt er direkte tilgjengelig på sitt eget website-endepunkt. Det er pedagogisk enkelt, men avviker fra anbefalt praksis i produksjon.

I en produksjonssetting gjøres bucketen privat, og CloudFront gis eksklusiv lesetilgang gjennom en **Origin Access Control (OAC)** — etterfølgeren til den eldre Origin Access Identity (OAI). Trafikken rutes da mot S3 sitt **REST-endepunkt** i stedet for website-endepunktet, bucket-policyen begrenses til CloudFront sin service-principal, og `aws_s3_bucket_public_access_block` settes tilbake til å blokkere all offentlig tilgang. Dette sikrer at innhold kun kan hentes gjennom CloudFront, slik at tilgangskontroll, logging og caching ikke kan omgås.

Avveiningen er økt kompleksitet: REST-endepunktet utfører ikke website-endepunktets automatiske oppslag av `index.html` for underkataloger, så ruting for en single-page application må håndteres eksplisitt — typisk med `custom_error_response`-regler eller en CloudFront Function. Fordi dette introduserer en annen origin-arkitektur og flere ressurser, er det utelatt fra bonusoppgaven, men nevnes her som det anbefalte sikkerhetsmønsteret for reelle driftsmiljøer.

## Oppsummering

Du har nå deployet og håndtert en statisk nettside på AWS ved hjelp av Terraform og AWS CLI.

## Neste steg

Når denne delen er på plass, fortsetter du med modulene, remote state og CI/CD-pipelinen i oppfølgeren:
[glennbechdevops/modules-remote-state](https://github.com/glennbechdevops/modules-remote-state).
