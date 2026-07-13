# 🛡️ Enterprise IT Lab: Bezpieczna Architektura Hybrydowa (MikroTik + Active Directory)

## 📌 Cel Projektu
Projekt symuluje bezpieczne środowisko firmowe (Enterprise) zbudowane od zera. Celem laboratorium jest wdrożenie zaawansowanej segmentacji sieci, zabezpieczenie routera brzegowego (Jump Box) oraz implementacja modelu Enterprise Access Model (Tiering) w środowisku Active Directory w celu obrony przed atakami typu Pass-the-Hash.

---

## 🗺️ Topologia Sieci



Środowisko zostało zwirtualizowane za pomocą Hyper-V i opiera się na dwóch wyizolowanych podsieciach (VLAN):

| Strefa (VLAN) | Adresacja IP | Brama Domyślna | Rola w projekcie |
| :--- | :--- | :--- | :--- |
| **VLAN 20 (Serwery)** | `192.168.20.0/24` | `192.168.20.1` | Strefa Tier 0/1. Zawiera Kontroler Domeny (Windows Server). Z tej strefy zarządzany jest router. |
| **VLAN 30 (Klienci)** | `192.168.30.0/24` | `192.168.30.1` | Strefa Tier 2. Zawiera stacje robocze (Windows 10). Brak bezpośredniego dostępu do infrastruktury sieciowej. |
| **WAN** | DHCP | DHCP | Wyjście na świat (Internet NAT). |

---

## 🚀 Etap 1: Architektura Sieciowa i Routing (Zakończone)
W pierwszej fazie projektu skonfigurowano fundamenty sieciowe, skupiając się na izolacji i bezpieczeństwie:

* **Trunking w Hyper-V:** Skonfigurowano wirtualny przełącznik w trybie Trunk za pomocą PowerShell (`Set-VMNetworkAdapterVlan`), aby przepuścić tagowany ruch (802.1Q) do wirtualnego routera MikroTik CHR.
* **Firewall (Łańcuch FORWARD):** Wdrożono politykę "Default Drop". Zezwolono jedynie na ruch z VLAN 30 do Internetu oraz celowy wyjątek do serwera AD (`192.168.20.10`).
* **Hardening Routera (Łańcuch INPUT):** Odcięto dostęp do usług zarządzania routerem (Winbox/SSH) ze wszystkich podsieci z wyjątkiem statycznego IP Windows Servera (koncepcja Jump Box).
* **Izolacja Warstwy 2 (MAC Server):** Zablokowano wykrywanie sąsiadów (MNDP) oraz połączenia po MAC adresie dla sieci klienckiej za pomocą MikroTik Interface Lists.

---

## 🛠️ Etap 2: Active Directory i Przejęcie DNS (Zakończone)
System Windows Server został wypromowany do roli Kontrolera Domeny (DC), stając się sercem uwierzytelniania w sieci.

* **Integracja DNS:** Zmieniono konfigurację DHCP w MikroTiku. Klienci (VLAN 30) pobierają teraz adres Kontrolera Domeny (`192.168.20.10`) jako swój jedyny serwer DNS.
* **DNS Forwarding:** Skonfigurowano Windows Server do rozwiązywania nazw lokalnych dla domeny oraz przekazywania zapytań o domeny zewnętrzne (np. google.com) w świat.
* **Struktura OU pod Tiering:** Zbudowano dedykowaną strukturę jednostek organizacyjnych (OU: Tier0, Tier1, Tier2) przygotowującą środowisko pod polityki bezpieczeństwa GPO.
* **Dołączenie stacji roboczej:** Klient Windows 10 został pomyślnie podłączony do domeny i umieszczony w odpowiednim kontenerze (Tier 2).

## 🔒 Etap 3: Bezpieczeństwo Urządzeń Końcowych i Serwera Plików (Zakończone)
Trzecia faza skupiła się na obronie przed atakami na poświadczenia lokalne oraz rygorystycznym zarządzaniu danymi biznesowymi.

* **Wdrożenie LAPS (Local Administrator Password Solution):** Rozszerzono schemat Active Directory i wdrożono automatyczną rotację haseł dla kont wbudowanych administratorów na stacjach roboczych. Hasła są bezpiecznie przechowywane w atrybucie `ms-Mcs-AdmPwd` i odczytywane wyłącznie przez autoryzowany personel IT.
* **Serwer Plików z zasadą AGDLP:** Zbudowano bezpieczną strukturę katalogów (np. Repozytorium IT, Faktury) opartą na gniazdowaniu grup, rozdzielając logicznie użytkowników (Role Biznesowe) od dostępu do zasobu (Uprawnienia NTFS).
* **Targetowanie Zasad Grupy (Item-Level Targeting):** Skonfigurowano GPO odpowiedzialne za mapowanie dysków sieciowych, wdrażając inteligentne warunki – pracownicy widzą wyłącznie dyski dedykowane dla ich działów (np. tylko członkowie grupy "Księgowość" widzą zmapowany dysk `K:\`).
* **Audytowanie Zdarzeń Zabezpieczeń (Informatyka Śledcza):** Włączono zaawansowane zasady inspekcji dostępu do obiektów w GPO. System plików generuje teraz precyzyjne logi (Event ID 4663 oraz 4660) wskazujące kto, z jakiego urządzenia i o jakiej godzinie usunął lub zmodyfikował wrażliwy plik firmowy.