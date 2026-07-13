## 📚 Słowniczek i Zagadnienia Teoretyczne
W projekcie wykorzystano następujące technologie i koncepcje architektoniczne:

### 1. Wirtualizacja i Warstwa 2 (L2)
* **Hyper-V Virtual Switch & Trunking:** Domyślnie wirtualne przełączniki w Hyper-V odrzucają ruch ze znacznikami VLAN. Użycie polecenia PowerShell `Set-VMNetworkAdapterVlan -Trunk` było konieczne, aby wirtualna karta sieciowa routera mogła przyjmować ruch z wieloma tagami (VLAN 20 i 30) na jednym fizycznym/wirtualnym porcie.
* **VLAN (Virtual Local Area Network - 802.1Q):** Logiczny podział jednej fizycznej sieci na mniejsze, odizolowane domeny rozgłoszeniowe (Broadcast Domains). W tym labie użyte do fizycznej separacji serwerów (Tier 0/1) od stacji roboczych (Tier 2).

### 2. Routing i Zapora Sieciowa (MikroTik RouterOS)
* **Brama Domyślna (Default Gateway):** Router MikroTik posiada wirtualne interfejsy z adresami IP przypisanymi do każdego VLAN-u (np. `192.168.20.1` dla VLAN 20). Pełnią one rolę punktu wyjścia (bramy) dla urządzeń w tych sieciach.
* **NAT / Masquerade:** Translacja adresów sieciowych. Pozwala ukryć prywatne adresy IP (192.168.x.x) naszych maszyn wirtualnych za jednym adresem przyznanym routerowi na interfejsie WAN, umożliwiając im dostęp do Internetu.
* **Firewall - Łańcuch FORWARD:** Analizuje pakiety, które przechodzą *PRZEZ* router (np. z VLAN 30 do VLAN 20 lub do Internetu). Skonfigurowany w modelu "Default Drop" (odrzucaj wszystko, co nie jest jawnie dozwolone).
* **Firewall - Łańcuch INPUT:** Analizuje pakiety, które są adresowane *DO* samego routera (np. próba logowania do panelu Winbox, pingowanie bramy).
* **MAC Server:** Specyficzna usługa w RouterOS działająca w Warstwie 2 (pomijająca IP Firewall). W projekcie została ograniczona wyłącznie do interfejsu VLAN 20, blokując ataki i próby logowania ze strony klientów na poziomie adresu MAC.

### 3. Tożsamość i Zarządzanie (Microsoft)
* **Jump Box (Bastion Host):** Koncepcja bezpieczeństwa polegająca na odcięciu bezpośredniego dostępu do urządzeń infrastrukturalnych (routerów, przełączników) z sieci użytkowników. Zarządzanie odbywa się wyłącznie z dedykowanego, zabezpieczonego serwera w strefie zarządzania.
* **Active Directory Domain Services (AD DS):** Usługa katalogowa, która centralizuje zarządzanie tożsamościami (użytkownikami, komputerami) i uwierzytelnianiem w sieci.
* **DNS (Domain Name System):** W środowisku domenowym DNS przestał być obsługiwany przez router. Windows Server (jako Kontroler Domeny) przejął rolę głównego serwera DNS, co jest niezbędne do poprawnego lokalizowania usług AD przez stacje robocze.
* **OU (Organizational Unit - Jednostka Organizacyjna):** Kontenery w Active Directory, w których grupujemy użytkowników i komputery. Pozwalają na aplikowanie konkretnych Zasad Grupy (GPO).
* **Enterprise Access Model (dawniej Model Tiering):** Model projektowania struktury AD chroniący przed atakami *Pass-the-Hash* i *Lateral Movement*. Polega na logicznym i fizycznym odseparowaniu uprawnień w trzech warstwach (Tier 0 - Kontrolery Domeny, Tier 1 - Serwery, Tier 2 - Stacje robocze i użytkownicy).

### 4. Rozwiązanie LAPS (Obrona przed Pass-the-Hash)
Atak *Pass-the-Hash* polega na wykradnięciu skrótu (hasha) hasła z pamięci RAM skompromitowanego komputera. Jeśli w firmie wszystkie komputery mają to samo hasło lokalnego administratora, atakujący może użyć tego jednego hasha do przejęcia każdej innej maszyny w sieci (tzw. *Lateral Movement*). LAPS całkowicie to eliminuje, zmuszając każdą maszynę do posiadania unikalnego, 20-znakowego, automatycznie zmieniającego się hasła, które jest bezpiecznie archiwizowane w kontrolerze domeny.

### 5. Model AGDLP
Złoty standard Microsoftu w zarządzaniu uprawnieniami do plików w dużych środowiskach. Zapobiega powstawaniu bałaganu autoryzacyjnego (tzw. "Permission Creep").
* **A**ccount (Konto) — np. `jkowalski`.
* **G**lobal Group (Grupa Globalna) — reprezentuje rolę w firmie, np. `G-Księgowość`.
* **D**omain Local Group (Grupa Lokalna) — reprezentuje uprawnienie do zasobu, np. `DL-Faktury-Modyfikacja`.
* **P**ermissions (Uprawnienia NTFS) — faktyczne uprawnienia przypisane do folderu.
Konta wpadają do G, G wpada do DL, a DL otrzymuje P. Administrator nigdy nie nadaje uprawnień bezpośrednio użytkownikowi.

### 6. Uprawnienia NTFS vs Uprawnienia Udziału (Share Permissions)
Podczas udostępniania folderu w sieci, Windows sprawdza dwie warstwy zabezpieczeń. Uprawnienia Udziału (sieciowe drzwi do pokoju) oraz Uprawnienia NTFS (zabezpieczenia samego pliku/sejfu). System zawsze wybiera **bardziej restrykcyjne** z tych dwóch. Powszechną praktyką inżynieryjną jest ustawienie uprawnień Udziału na "Wszyscy - Pełna Kontrola" i sterowanie faktycznym dostępem wyłącznie na bardzo precyzyjnej warstwie NTFS (Zabezpieczenia).

### 7. Event ID 4663 i 4660 (Windows Security Auditing)
Kluczowe identyfikatory zdarzeń wykorzystywane przez zespoły SOC (Security Operations Center):
* **4663:** Próba uzyskania dostępu do obiektu. W szczegółach (wartość `Accesses`) zdradza, czy użytkownik chciał zapisać dane (`WriteData`) czy je usunąć (`DELETE`). Często generuje dużo logów.
* **4660:** Obiekt został fizycznie usunięty. To niepodważalny dowód kasacji pliku lub podfolderu. Zawsze występuje w parze z odpowiednim logiem 4663.