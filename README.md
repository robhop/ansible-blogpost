# Ansible - konfigurering av maskiner for programmerere?

Det finnes et utall datasystemer som har som hensikt å administrere datamaskiner, Ansible er ett av dem, og kanskje passer det litt bedre for de som i utgangspunktet er vant til å skrive kode.

Ansible installeres på en kontrollmaskin og denne brukes til å administrere andre maskiner. Kontrollmaskinen må være en UNIX variant (OS X inkludert), men Ansible kan brukes til å administrere Windows maskiner også, dog ikke med like stor grad av enkelhet som andre UNIX maskiner. 

Jeg vil i denne demoen vise hvordan Ansibl kan brukes til å konfigurere to servere fra en kontrollmaskin. Jeg valgte å kjøre 3 Ubuntu 16.04 installasjoner i Virtual box, alle med en bruker som heter robert og ssh server på standard port 22.
 
## Installasjon

### Kontrollmaskin

Som nevnt tidligere trenger bare Ansible å installeres på kontrollmaskinen, siden min kontrollmaskin kjører Ubuntu 16.04 gjøres det slik:
	
	$ sudo apt-get update
	$ sudo apt-get install software-properties-common
	$ sudo apt-add-repository ppa:ansible/ansible
	$ sudo apt-get update
	$ sudo apt-get install ansible

Ansible dokumentasjonen gir god veiledning for installasjon på andre plattformer.

### Servere

Maskinene som kontrolleres av kontrollmaskinen (Servere) trenger ikke noe egen programmvare fra ansible, men ssh-server og pyton må være installert. Vanligvis kommer dette med ubuntu, men for å være sikker kan man kjøre følgenede kommandoer på server1 og server2.

	$ sudo apt-get update
	$ sudo apt-get install openssh-server
	$ sudo apt-get install python-minimal

### Pålogging med SSH-nøkler

Det er sterkt anbefalt å bruke ssh nøker for kommunikasjon mellom kontrollmaskin og maskiner som kontrolleres. Dersom bruker med samme navn finnes på maskine du vil kontrollere fra kontrollmaskin kan ssh nøkkel enkel kopiers, først sjekk at du har en ssh nøkkel:

	$ ssh-keygen

Finnes nøkkelen fra før får du beskjed om det, så kopierer du nøkkelen til ønsket maskin, i mitt tilfelle 10.0.2.4 og 10.0.2.5.

	$ ssh-copy-id robert@10.0.2.4
	$ ssh-copy-id robert@10.0.2.5

Nå kan ansible logge på maskine den skal fjernstyre med brukeren robert, uten å måtte skrive inn noe passord.

## Vårt første ansible program

Ansible består av en rekke kommandolinje-program, det enkelste heter **ansible** og kan brukes til å kjøre enkele kommandoer på en eller flere maskiner.

Kommandostrukturen for ansible er

	ansible <host-pattern> [options]

Localhost er definert som kontrollmaskine og kan bruke som `<host-pattern>`.
Ansible kommer med et stort sett [**moduler**](https://docs.ansible.com/ansible/latest/user_guide/modules_intro.html), en av de enkelste heter [*ping*](https://docs.ansible.com/ansible/latest/modules/ping_module.html) og bruke til å pingen en maskin og se om den svarer.

Ping localhost og se om den svarer:

	$ ansible localhost -m ping
	localhost | SUCCESS => {
    	"changed": false,
    	"ping": "pong"
	}

Å pinge seg selv er ikke akkurat revolusjonerende, for å nå andre maskiner må vi definere litt **inventar**.

### Inventar

Først litt husarbeid, Ansible leter etter konfigurasjonsfiler i gjeldende mappe og det er derfor lurt å lage en egen mappe til demoprosjektet.
	
	$ mkdir ansible-demo
	$ cd ansible-demo/


Så definerer vi vår infrastruktur eller «inventory» som det kalles. Det kan gjøres i en fil som vi kaller for hosts. Vi kan brukre ini-fil struktur eller YAML, her er et eksempel som bruker ini-fil struktur.

	[servere]
	server1 ansible_host=10.0.2.4
	server2 ansible_host=10.0.2.5

Her definerer vi en gruppe som vi kaller *servere* hvor våre to maskiner er medlemmer, vi oppgir også ip addressen til maskinen i hosts-variabelen *ansible_host*.

Vi forteller ansible at inventaret vårt finnes i filen hosts ved å bruke `-i` opsjonen, og bruker `<host-pattern>` `servere` for å kjøre ansible-modulen `ping`. Ansible vil da pinge alle server i gruppen `servere`.

	$ ansible servere -i hosts -m ping
	server2 | SUCCESS => {
	    "changed": false,
	    "ping": "pong"
	}
	server1 | SUCCESS => {
	    "changed": false,
	    "ping": "pong"
	}

Vi kan også spesifiser hver enkelt maskin for seg selv, eller bruke det pre-definerte `<host-pattern> all`

	$ ansible server1 -i hosts -m ping
	server1 | SUCCESS => {
	    "changed": false,
	    "ping": "pong"
	}
	$ ansible server2 -i hosts -m ping
	server2 | SUCCESS => {
	    "changed": false,
	    "ping": "pong"
	}
	$ ansible all -i hosts -m ping
	server1 | SUCCESS => {
	    "changed": false,
	    "ping": "pong"
	}
	server2 | SUCCESS => {
	    "changed": false,
	    "ping": "pong"
	}


### Faktsik gjøre noe:

Vi kan nå kjøre en kommando som faktisk gjør noe. F.eks. installere *Network Time Protocol (NTP) daemon* ved hjelp av apt.
For å gjøre dette på våre to servere kan vi kjøre kommandoen.

	$ ansible servere -m "apt" -a "name=ntp state=present" -i hosts --become --ask-become-pass

Vi forteller her ansible følgende:

* Utføre handlingen på maskiner som passer `<host-pattern> servere` .
* Kjør modulen [apt](https://docs.ansible.com/ansible/latest/modules/apt_module.html) `-m "apt"`
* Gi modulen parameterene `name=ntp state=installed`, dvs: *se til at pakken ntp er installert*.
* Bruk inventar filen hosts `-i hosts`
* Utfør handingen som root `--become`
* Spør etter root passord `--ask-become-pass`

Vel å merke seg er at ansible moduler prøver å sette utføre en endring slik at en tilstand oppnås, kommandoer kan derfor utføres flere ganger. Dersom ntp pakken allerede finnes på servereene vil ansible svare **SUCCESS**.

Vi kan også endre tilstanden fra *present* til *absent* og dermed avinstallere pakken.

Prøv dette med:

	$ ansible servere -m "apt" -a "name=ntp state=absent" -i hosts --become --ask-become-pass

Men Ansible skinner først når vi lager en playbook og kjører den med `ansible-playbook`.

## Vår første ansible playbook

[ansible-playbook](https://docs.ansible.com/ansible/latest/user_guide/playbooks_intro.html) tar som parameter en yaml fil som definere oppgaver som skal utføres.

For å gjøre samme oppgave som vi gjorde med `ansible` i en playbook kan vi lage en fil som heter `servere_setup.yaml`:

	---
	- hosts: servere
	  tasks:
	  - name: Install ntp deamon
	    become: true
	    apt:
	      update_cache: true
	      name: ntp
	      state: absent

For å kjøre denne filen med ansible-playbook:

	$ ansible-playbook servere_setup.yaml -i hosts --ask-become-pass

**ansible-playbook** gjør en mer omstendelig jobb enn **ansible**, den henter inn fakta om maksinene våre, utfører komandoer og gir en oversikt over potensielle endringer som er utført.





	---
	- hosts: servere
	  tasks:
	  - name: Install ntp deamon
	    become: true
	    apt:
	      update_cache: true
	      name: ntp
	      state: absent