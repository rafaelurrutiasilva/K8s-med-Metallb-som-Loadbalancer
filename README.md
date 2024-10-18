### [🇸🇪 Svenska](README.md) / [🇬🇧 English](README_en.md)

# Kubernetes kluster med Metallb som Lastbalanserare
## Introduktion

### Bakgrund

Jag brukar ofta skapa en komplett labbmiljö på min egen laptop med hjälp av VMware Workstation, där jag sätter upp olika virtuella maskiner för att genomföra testprojekt eller förverkliga idéer. Nu har jag fått möjlighet att sätta upp en fristående VMware ESXi-server och blev därför *tvungen* att bygga ett Kubernetes-kluster samt konfigurera en *LoadBalancer* istället för att använda `kubectl port-forward service`.

### Sammanfattning

Att enbart följa en av dessa artiklar skulle inte ha hjälpt mig att uppnå mitt mål, vilket var att exponera de tjänster jag skapar i mitt Kubernetes-kluster externt via en lastbalanserare.

Här har jag sammanställt de artiklar jag använde, tillsammans med en kort beskrivning av hur jag gick tillväga. Allt är främst skrivet för att hjälpa mitt glömska jag att komma ihåg processen – men om det kan vara till nytta för dig, så varsågod och använd det!

###  Labbmiljö 

Labbmiljön bestod av ett Kubernetes-kluster med tre noder, där varje nod var en virtuell maskin som körde Ubuntu 20.04. Allt detta drevs på en fristående ESXi-server, version 8.
### Uppdatering
<aside>
💡 Det fungerade också bra med VMware Workstation 17 Pro i Windows 10. Det som krävdes var återigen att adress-poolen i LBs konfiguration inte kolliderade med vad DHCP ( i VMware Workstation) delade ut.
</aside>

### Referenser

 https://metallb.universe.tf/installation

https://blog.andreev.it/2023/10/install-metallb-on-kubernetes-cluster-running-on-vmware-vms-or-bare-metal-server

https://akyriako.medium.com/load-balancing-with-metallb-in-bare-metal-kubernetes-271aab751fb8

# **Handledning**

## Installation och konfiguration

Installationen utgår ifrån det som publiceras på hemsidan och fortsätter sen med det som framgår av artikeln 2 i listan. Vi att konfigurera kube-proxy med det som behövs samt konfigurera MetalLB med en loadbalancer med det IP adress-pool jag behöver.  

### Enable strict ARP mode

Om du använder kube-proxy i IPVS-läge, måste du sedan Kubernetes v1.14.2 aktivera strict ARP mode. Du kan uppnå detta genom att redigera kube-proxy-konfigurationen i det aktuella klustret.

```bash 
kubectl get configmap kube-proxy -n kube-system -o yaml \
 |sed -e "s/strictARP: false/strictARP: true/" | kubectl apply -f - -n kube-system
```

> Obs! Detta inte behövs om du använder kube-router som service-proxy, eftersom det aktiverar strict ARP som standard.
> 

### Installation enligt metallb.universe.tf

Detta kommer att installera MetalLB i ditt kluster, under namespace `metallb-system`.

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.8/config/manifests/metallb-native.yaml
```

**Objekt i manifestet är:**

- `metallb-system/controller` deployment. Detta är den klusterövergripande kontrollenheten som hanterar IP-adresstilldelningar.
- `metallb-system/speaker` daemonset. Detta är komponenten som använder det/de protokoll du väljer för att göra tjänsterna nåbara.
- Servicekonton för controller och speaker, tillsammans med de RBAC-behörigheter som komponenterna har behövs för att fungera.

> Installationsmanifestet inkluderar inte någon konfigurationsfil. MetalLB komponenter kommer att starta, men förblir inaktiva tills du börjar distribuera resurser.
> 

### Kontrollerar status på Metallb bas-objekt

Du kan kontrollera att de olika objekt i manifestet är ingång utan fel.

```bash
kubectl -n metallb-system get pods
kubectl -n metallb-system get all
```

### Konfigurera LoadBalancer

Nu kan vi gå vidare och konfigurera en adress-pool till vår loadbancer. Hör får du använde det intervall som passar dig. Intervallet måste vara reserverat och inte användas av DHCP:en i din nät.

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - 172.29.8.170-172.29.8.180           # Använd det som passar i din miljö
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: example
  namespace: metallb-system   
```

Spara detta i en yaml-fil och kör sedan `kubectl apply -f <yaml-fil>` 

 

## Testa att LB gör sitt

Vi kommer att installera två enkla applikationer som vi sedan testar både internt från Worker Nodes och externt klient via en webbläsare. Dessa applikationer kommer vi att se vilken pod inom klustret som svarar på frågan, vilket hjälper oss att kontrollera att lastbalansering fungerar.

### App Demo1

Detta startar en lättviktig webbserver som kommer att visa namnet på poden där den körs.

```bash
kubectl create deployment demo1 --image=gcr.io/google-samples/hello-app:1.0 --replicas 3
kubectl expose deployment demo1 --type LoadBalancer --port 80 --target-port 8080
```

### App Demo2

```bash
kubectl create deployment demo2 --image=klimenta/serverip --replicas 6 --port 3000
kubectl expose deployment demo2 --type LoadBalancer --port 80 --target-port 3000
```

### Testa Demo1 & Demo1

Du kan köra följande flera gånger i cli eller surfa in på den URL som visas.

```bash
# demo1
curl $(kubectl get svc demo1 |awk '/^demo1/  {print"http://"$4}'); \
echo "Testa själv nu från din webbklient:"; \
echo $(kubectl get svc demo1 |awk '/^demo1/  {print"http://"$4}')

# demo2
curl $(kubectl get svc demo2 |awk '/^demo2/  {print"http://"$4}'); \
echo "Testa själv nu från din webbklient:"; \
echo $(kubectl get svc demo2 |awk '/^demo2/  {print"http://"$4}')
```

### Test med loop

Vill du testa med loop i cli? Använd detta exempelvis.

```bash
for i in {1..5}
do
 curl $(kubectl get svc demo1 |awk '/^demo1/ {print"http://"$4}') ; echo "" 
done
```

## Rensa allt

Detta kan göras på flera olika sätt, men gemensamt för alla är användningen av CLI-kommandona `kubectl delete service` och `kubectl delete deployment`.

```bash
for I in service/demo1 deployment.apps/demo1 service/demo2 deployment.apps/demo2 
do 
   kubectl delete $I
done
```
