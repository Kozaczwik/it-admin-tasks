# 🛡️ Enterprise IT Lab: Bezpieczna Architektura Hybrydowa (MikroTik + Active Directory)

## 📌 Cel Projektu
Projekt symuluje bezpieczne środowisko firmowe (Enterprise) zbudowane od zera. Celem laboratorium jest wdrożenie zaawansowanej segmentacji sieci, zabezpieczenie routera brzegowego (Jump Box) oraz implementacja modelu Enterprise Access Model (Tiering) w środowisku Active Directory w celu obrony przed atakami typu Pass-the-Hash.

---

## 🗺️ Topologia Sieci
Środowisko zostało zwirtualizowane za pomocą Hyper-V i opiera się na dwóch wyizolowanych podsieciach (VLAN):

| Strefa (VLAN) | Adresacja IP | Brama Domyślna | Rola w projekcie |
| :--- | :--- | :--- | :--- |
| **VLAN 20 (Serwery)** | `192.168.20.0/24` | `192.168.20.1` | Strefa Tier 0/1. Kontroler Domeny, zarządzanie routerem. |
| **VLAN 30 (Klienci)** | `192.168.30.0/24` | `192.168.30.1` | Strefa Tier 2. Stacje robocze, brak dostępu do infrastruktury. |
| **WAN** | DHCP | DHCP | Wyjście na świat (Internet NAT). |

---

## 🚀 Etap 1: Architektura Sieciowa i Routing (Zakończone)
W pierwszej fazie projektu skonfigurowano fundamenty sieciowe, skupiając się na izolacji i bezpieczeństwie:

* **Trunking w Hyper-V:** Skonfigurowano wirtualny przełącznik w trybie Trunk (`Set-VMNetworkAdapterVlan`) dla tagowanego ruchu (802.1Q) do MikroTik CHR.
* **Firewall (Łańcuch FORWARD):** Wdrożono politykę "Default Drop". Zezwolono jedynie na ruch z VLAN 30 do Internetu oraz celowy wyjątek do serwera AD (`192.168.20.10`).
* **Hardening Routera (Łańcuch INPUT):** Ograniczono dostęp do usług zarządzania (Winbox/SSH) wyłącznie do statycznego IP Windows Servera.
* **Izolacja Warstwy 2 (MAC Server):** Zablokowano wykrywanie sąsiadów (MNDP) i połączenia po MAC dla sieci klienckiej (Interface Lists).

---

## 🛠️ Etap 2: Active Directory i Przejęcie DNS (Zakończone)
System Windows Server został wypromowany do roli Kontrolera Domeny (DC).

* **Integracja DNS:** Klienci (VLAN 30) korzystają wyłącznie z serwera DNS na kontrolerze domeny (`192.168.20.10`).
* **DNS Forwarding:** Skonfigurowano rozwiązywanie nazw lokalnych oraz przekazywanie zapytań zewnętrznych.
* **Struktura OU pod Tiering:** Zaimplementowano model Tier 0, Tier 1, Tier 2 dla obiektów w AD.
* **Dołączenie stacji:** Klient Windows 10 został podłączony do domeny w kontenerze Tier 2.

---

## 🔒 Etap 3: Bezpieczeństwo Urządzeń Końcowych i Serwera Plików (Zakończone)
* **LAPS:** Wdrożono automatyczną rotację haseł lokalnych administratorów, przechowywanych bezpiecznie w atrybucie `ms-Mcs-AdmPwd`.
* **Serwer Plików (AGDLP):** Zbudowano strukturę opartą na zagnieżdżaniu grup, rozdzielając role biznesowe od uprawnień NTFS.
* **GPO (Item-Level Targeting):** Mapowanie dysków sieciowych z warunkami widoczności zależnymi od przynależności do grup.
* **Informatyka Śledcza:** Włączono audyt zdarzeń 4663 i 4660, monitorujący modyfikacje i usuwanie plików przez użytkowników.

---

## 🛡️ Etap 4: Infrastruktura PKI, Filtrowanie Ruchu i Automatyzacja (Zakończone)
* **AD CS (PKI):** Wdrożono korporacyjny Urząd Certyfikacji (Root CA) z pełnym Auto-Enrollmentem certyfikatów dla maszyn.
* **DNS Sinkholing:** Blokada serwisów społecznościowych poprzez kierowanie domen na `127.0.0.1`.
* **DNS Hijacking:** Przechwytywanie zapytań DNS (NAT dst-nat port 53) w celu wymuszenia korzystania z filtrowania OpenDNS.
* **Ochrona przed Malware:** Zautomatyzowany skrypt (Scheduler + fetch) importujący bazy złośliwych IP (np. CERT) do list adresowych firewalla.

---

## 🛡️ Etap 5: Hardening Systemowy i Monitoring (Zakończone)
* **Zaawansowany Audyt:** Wdrożono *Advanced Audit Policy* z logowaniem komend wiersza poleceń (Event 4688).
* **Mikrosegmentacja:** Zapora Windows Defender Firewall (GPO) blokuje komunikację poprzeczną (Lateral Movement) między stacjami roboczymi w VLAN 30.
* **Protected Users:** Dodano konta administracyjne do grupy *Protected Users*, wymuszając Kerberos i usuwanie poświadczeń z pamięci RAM.
* **Dezintegracja Legacy:** Wyłączono protokoły LLMNR oraz NetBIOS, by zapobiec atakom typu *Responder*.
* **Monitoring:** Zabezpieczono logi MikroTika poprzez trwały zapis na dysk oraz wysyłkę zdalną (Remote Syslog) na port UDP 514.