\# Firewall + DMZ: Segurança de Perímetro e Defense in Depth



Laboratório desenvolvido com Kathará para a disciplina de Segurança de Redes.



O objetivo é implementar uma arquitetura com LAN, DMZ, firewall e roteador de borda, aplicando princípios de segurança de perímetro, Default Deny, menor privilégio, filtragem stateful e Defense in Depth.



\## 1. Topologia



A topologia utilizada foi:



```text

&#x20;                        Internet

&#x20;                           |

&#x20;                           |

&#x20;                          r0

&#x20;                 Roteador de borda/NAT

&#x20;                           |

&#x20;                  198.51.100.0/30

&#x20;                           |

&#x20;                          fw

&#x20;                    /             \\

&#x20;                   /               \\

&#x20;         10.0.1.0/24             10.0.2.0/24

&#x20;             LAN                     DMZ

&#x20;          /       \\                /     \\

&#x20;        pc1       pc2            web     dns

```



\### Endereçamento



| Dispositivo | Interface | Endereço |

|---|---|---|

| r0 | eth0 | 198.51.100.2/30 |

| fw | eth0 | 198.51.100.1/30 |

| fw | eth1 | 10.0.1.1/24 |

| fw | eth2 | 10.0.2.1/24 |

| pc1 | eth0 | 10.0.1.10/24 |

| pc2 | eth0 | 10.0.1.11/24 |

| web | eth0 | 10.0.2.10/24 |

| dns | eth0 | 10.0.2.11/24 |



O `r0` também possui uma interface externa criada através do modo `bridged` do Kathará.



\## 2. Configuração inicial e validação



Antes da aplicação das regras restritivas do firewall, foram configurados o endereçamento IP, gateways, rotas e encaminhamento IPv4.



O `r0` funciona como roteador de borda e utiliza NAT/MASQUERADE para permitir a saída das redes internas para a Internet.



Foram validados:



\- comunicação entre LAN e DMZ;

\- comunicação da LAN com a Internet;

\- roteamento realizado pelo firewall;

\- servidor Web na DMZ;

\- servidor DNS na DMZ.



\### Servidor Web



O host `web` executa Apache na porta TCP/80.



A partir do PC1 foi realizado:



```bash

curl http://10.0.2.10

```



O servidor retornou a página padrão do Apache, comprovando o acesso LAN -> Web da DMZ.



\### Servidor DNS



O host `dns` utiliza BIND e encaminha consultas externas para `8.8.8.8`.



Foi configurada recursão para as redes do laboratório e localhost.



Teste realizado no PC1:



```bash

dig @10.0.2.11 google.com

```



A consulta retornou `status: NOERROR` e um endereço IPv4 para `google.com`, utilizando `10.0.2.11` como servidor DNS.



\## 3. Política de Segurança de Perímetro



O firewall foi configurado com `iptables`.



A política adotada foi \*\*Default Deny\*\* na chain `FORWARD`:



```bash

iptables -P FORWARD DROP

```



Dessa forma, tráfego encaminhado pelo firewall é bloqueado por padrão e apenas comunicações explicitamente necessárias são permitidas.



\### Filtragem stateful



Foi utilizado o módulo `conntrack`:



```bash

iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

```



Assim, respostas pertencentes a conexões previamente autorizadas podem retornar sem a necessidade de liberar indiscriminadamente o sentido contrário.



\### Política implementada



| Comunicação | Resultado |

|---|---|

| LAN -> Internet | Permitido |

| LAN -> Web da DMZ (TCP/80) | Permitido |

| LAN -> DNS da DMZ (UDP/TCP 53) | Permitido |

| DNS da DMZ -> DNS externo (UDP/TCP 53) | Permitido |

| Internet/WAN -> Web da DMZ (TCP/80) | Permitido |

| Internet/WAN -> LAN | Bloqueado |

| DMZ -> LAN para novas conexões | Bloqueado |

| ESTABLISHED/RELATED | Permitido |



A saída do servidor DNS para a Internet foi limitada às portas UDP/TCP 53, seguindo o princípio do menor privilégio.



\## 4. Publicação do servidor Web



No roteador de borda foi utilizado DNAT para encaminhar conexões HTTP recebidas pela interface externa para o servidor Web da DMZ:



```bash

iptables -t nat -A PREROUTING -i eth1 -p tcp --dport 80 -j DNAT --to-destination 10.0.2.10:80

```



No firewall, apenas o acesso TCP/80 destinado ao servidor `10.0.2.10` foi autorizado.



O acesso externo foi testado utilizando um container conectado à rede externa do Docker:



```bash

docker run --rm curlimages/curl:latest http://172.17.0.2

```



O teste retornou a página padrão do Apache, comprovando o caminho:



```text

Cliente externo -> r0 -> DNAT -> fw -> DMZ -> Web

```



Também foi realizado um teste do lado WAN para a LAN:



```bash

ping -c 3 10.0.1.10

```



O resultado foi 100% de perda de pacotes, confirmando que novas conexões WAN -> LAN são bloqueadas pela política Default Deny.



Da mesma forma, uma tentativa iniciada pelo servidor Web da DMZ em direção ao PC1 apresentou 100% de perda, demonstrando o bloqueio de novas conexões DMZ -> LAN.



\## 5. Experimentos por camada



\### 5.1 Camada 2 - Bloqueio por MAC



O experimento proposto consiste em representar o PC2 como uma máquina comprometida e bloquear seu endereço MAC no firewall.



A regra pode ser implementada utilizando o módulo `mac` do iptables, por exemplo:



```bash

iptables -I FORWARD 2 -i eth1 -m mac --mac-source <MAC\_DO\_PC2> -j DROP

```



Procedimento:



```text

Antes da regra:

PC1 -> tráfego permitido

PC2 -> tráfego permitido



Regra:

bloqueio do endereço MAC do PC2 no firewall



Depois da regra:

PC1 -> permanece permitido

PC2 -> bloqueado

```



O endereço MAC pertence à Camada 2 e não acompanha o pacote por todo o caminho na Internet. A cada salto realizado por um roteador, o cabeçalho Ethernet é reconstruído.



Neste laboratório, o firewall consegue observar o MAC original do PC2 porque PC2 e a interface LAN do firewall pertencem ao mesmo segmento de Camada 2. Caso existisse outro roteador entre eles, o firewall observaria o MAC desse próximo salto, e não diretamente o MAC original do PC2.



> Este experimento foi planejado, mas não foi concluída a coleta de evidências antes da entrega.



\### 5.2 Camada 3 - ICMP



Para demonstrar filtragem em Camada 3, pode-se bloquear ICMP entre redes.



Exemplo de regra:



```bash

iptables -I FORWARD 2 -i eth1 -o eth0 -p icmp -j DROP

```



O ciclo de validação consiste em executar um `ping` antes da regra, aplicar o bloqueio e repetir o teste.



O `tcpdump` pode ser utilizado para observar a passagem ou o bloqueio dos pacotes ICMP nas interfaces do firewall.



> Este experimento foi planejado, mas não foi concluída a coleta de evidências antes da entrega.



\### 5.3 Camada 3 - Bloqueio por IP externo



Como a LAN possui acesso à Internet, um destino externo específico pode ser bloqueado pelo endereço IP:



```bash

iptables -I FORWARD 2 -i eth1 -o eth0 -d <IP\_DESTINO> -j DROP

```



O bloqueio exclusivamente por IP possui limitações. Um domínio pode utilizar vários endereços IP, endereços podem mudar dinamicamente e serviços diferentes podem compartilhar a mesma infraestrutura, como ocorre com CDNs e hospedagem compartilhada.



> Este experimento foi planejado, mas não foi concluída a coleta de evidências antes da entrega.



\### 5.4 Camada 4 - Portas TCP/UDP



Para representar o bloqueio de aplicações P2P/BitTorrent, podem ser utilizadas portas TCP/UDP escolhidas para o experimento.



Exemplo utilizando a porta 6881:



```bash

iptables -I FORWARD 2 -i eth1 -p tcp --dport 6881 -j DROP

iptables -I FORWARD 2 -i eth1 -p udp --dport 6881 -j DROP

```



O bloqueio somente por portas não é suficiente para identificar uma aplicação de forma confiável, pois aplicações podem utilizar portas alternativas ou dinâmicas. Controles em camadas superiores podem ser necessários para identificar o tráfego independentemente da porta utilizada.



> Este experimento foi planejado, mas não foi concluída a coleta de evidências antes da entrega.



\## 6. Controle de Camada 7



Como possibilidade de controle de Camada 7, foi escolhido o \*\*DNS Filtering\*\*.



Nesse modelo, o servidor DNS pode aplicar políticas sobre os nomes consultados, permitindo ou bloqueando determinados domínios ou categorias.



Diferentemente de uma regra baseada somente no endereço IP ou porta, o DNS Filtering permite aplicar políticas relacionadas aos nomes utilizados pelos clientes.



Por exemplo, uma consulta para um domínio proibido poderia ser bloqueada pelo resolvedor DNS, enquanto domínios autorizados continuariam sendo resolvidos normalmente.



Esse controle deve ser utilizado como uma camada adicional de segurança, e não como substituto do firewall e da segmentação da rede.



\## 7. Defense in Depth



A DMZ foi utilizada para separar serviços que precisam receber acessos externos da rede interna.



Caso o servidor Web da DMZ seja comprometido, isso não significa que o atacante terá acesso direto aos computadores PC1 e PC2.



No laboratório, uma nova conexão iniciada pela DMZ em direção à LAN é bloqueada pela política Default Deny do firewall.



As principais camadas de proteção são:



\- separação entre LAN e DMZ;

\- firewall entre as redes;

\- política Default Deny;

\- princípio do menor privilégio;

\- filtragem stateful;

\- limitação das comunicações permitidas;

\- controles aplicados aos próprios hosts e serviços.



Assim, o comprometimento de uma camada não deve resultar automaticamente no comprometimento de toda a infraestrutura. A segmentação reduz o movimento lateral e limita o impacto de um possível ataque.



\## 8. Respostas finais



\### Quem pode se comunicar com quem?



A LAN pode acessar a Internet e os serviços Web/DNS autorizados na DMZ. O lado externo pode acessar apenas o serviço Web publicado. Novas conexões originadas da DMZ para a LAN e da WAN para a LAN são bloqueadas.



\### Quais tipos de comunicação são permitidos ou bloqueados?



São permitidos apenas os fluxos necessários definidos na política do firewall. Respostas de conexões autorizadas são permitidas através de `ESTABLISHED,RELATED`. Os demais fluxos encaminhados são bloqueados pela política `DROP`.



\### Se uma camada falhar, quais outras continuam protegendo?



A arquitetura utiliza múltiplas camadas. Por exemplo, mesmo que o servidor Web seja comprometido, a segmentação da DMZ e o firewall continuam limitando novas conexões em direção à LAN. Controles nos hosts e serviços representam camadas adicionais de proteção.



\## 9. Execução



Na pasta do laboratório:



```bash

kathara lstart

```



Para encerrar:



```bash

kathara lclean

```



Os arquivos `.startup` configuram automaticamente os dispositivos, rotas, serviços e regras necessárias.

