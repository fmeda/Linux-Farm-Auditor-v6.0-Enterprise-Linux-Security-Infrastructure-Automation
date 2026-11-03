# MANUAL DO USUÁRIO — LINUX FARM AUDITOR v6.0
**Edição operacional PT-BR | 03/11/2026 | Público:** administradores Linux N2/N3, NOC, SOC, SRE e responsáveis pelo Ansible Controller.

> **Escopo:** este manual documenta o comportamento verificado no código entregue, não uma certificação CIS/NIST. A solução é predominantemente read-only quanto à configuração dos hosts, mas cria arquivos temporários nos hosts e relatórios no Controller.

## 1. Finalidade e limites
O Linux Farm Auditor dispara uma auditoria de múltiplos servidores via Ansible, coleta evidências por SSH e grava os relatórios em um compartilhamento **já montado** no controlador. Inclui métricas de saúde, verificações básicas de segurança, inventário de pacotes e correlação **offline** com avisos CVE informados pelo operador. **Não** corrige configurações, instala patches, executa exploits, consulta feeds online automaticamente ou comprova ausência de vulnerabilidades.

## 2. Arquitetura e fluxo
1. Operador inicia `run.sh` no Controller.
2. `run.sh` valida inventário, caminhos de entrada, Ansible e ponto de montagem; cria um diretório único de execução com `umask 077`.
3. `audit.yml` acessa cada host do grupo `linux_farm`, envia três coletores para diretório temporário, executa auditoria e inventário, lê `/etc/os-release` e recupera arquivos via `ansible.builtin.fetch`.
4. O Controller gera consolidação HTML/CSV/JSON com `consolidate.py`.
5. Se informado um JSON de avisos, executa `correlate.py`, enriquecendo opcionalmente com snapshots KEV e EPSS.
6. O playbook tenta remover o diretório temporário em `always`. Uma queda abrupta de SSH pode impedir a limpeza.

**Fronteira de confiança:** somente o Controller precisa de acesso de escrita à pasta NFS/SMB; os servidores auditados não precisam montá-la.

## 3. Componentes do pacote
| Arquivo | Função |
|---|---|
| `run.sh` | Entrada principal; validações, orquestração e códigos de saída |
| `inventory.ini` | Hosts Ansible, grupo `[linux_farm]` |
| `audit.yml` | Playbook de coleta, `block/rescue/always` e `fetch` |
| `linux_auditor.sh` | Auditoria de métricas, logs e verificações locais |
| `security_report.py` | Relatórios estruturados de segurança |
| `package_inventory.py` | Inventário local RPM/dpkg |
| `consolidate.py` | Consolidação de hosts, checks, CVEs importadas e HTML |
| `correlate.py` | Correlação offline com avisos de fabricantes, KEV e EPSS |
| `SECURITY.md` | Modelo de confiança, riscos e alinhamentos |
| `tests/` | Testes unitários de consolidação e escopo de CVE |

## 4. Requisitos e preparação
**Controller:** Linux, Bash, Python 3, `ansible-playbook`, `mountpoint`, `realpath`, SSH funcional, permissões de gravação na pasta de rede e acesso ao inventário. Para comparação de versões Debian, instale `dpkg` no Controller; para RPM, `rpmdev-vercmp`. Ausência de comparador gera `UNKNOWN_COMPARATOR`, não um achado confirmado.

**Hosts:** SSH, Bash, Python 3, ferramentas Linux comuns e permissões suficientes para as verificações desejadas. O playbook usa `become: false`; portanto, verificações que exigem root podem retornar dados incompletos. Prepare uma conta de auditoria dedicada, com privilégio mínimo e autorização explícita para leitura.

**Verificação prévia:**
```bash
bash -n run.sh linux_auditor.sh
python3 -m unittest discover -s tests -v
ansible --version
ansible-inventory -i inventory.ini --graph
ansible -i inventory.ini linux_farm -m ping
mountpoint -q /mnt/audit-share && test -w /mnt/audit-share && echo "Destino pronto"
```
O `ansible -m ping` valida conexão e Python remoto; não é um ping ICMP.

## 5. Configuração de inventário
Exemplo:
```ini
[linux_farm]
srv-rhel01 ansible_host=192.0.2.10 ansible_user=audituser
srv-ubuntu02 ansible_host=192.0.2.11 ansible_user=audituser

[linux_farm:vars]
ansible_ssh_common_args=-o BatchMode=yes
```
Os endereços `192.0.2.x` são apenas exemplos de documentação. Use nomes de host de inventário compostos por letras, números, ponto, hífen ou sublinhado. O consolidado identifica hosts a partir de entradas INI simples no grupo `[linux_farm]`; inventários dinâmicos/YAML não têm detecção completa de hosts ausentes nessa implementação. Configure verificação de host keys no SSH e não desative `StrictHostKeyChecking`.

## 6. Compartilhamento NFS ou SMB
O caminho informado deve ser **absoluto, existente, gravável e um mountpoint**. Não passe um diretório comum ou caminho UNC diretamente ao `run.sh`.

Exemplo NFS (a montagem depende de autorização e configuração local):
```bash
sudo mkdir -p /mnt/audit-share
sudo mount -t nfs4 nas.exemplo.local:/exports/linux-audit /mnt/audit-share
mountpoint /mnt/audit-share
```
Exemplo SMB/CIFS:
```bash
sudo mkdir -p /mnt/audit-share
sudo mount -t cifs //fileserver.exemplo.local/linux-audit /mnt/audit-share \
  -o credentials=/etc/linux-audit/smb-credentials,vers=3.1.1
```
Proteja o arquivo de credenciais com permissões restritas e não inclua senhas em scripts, histórico de shell ou inventário. Configure ACLs no servidor de arquivos, retenção, backups e monitoramento do compartilhamento. A montagem pode falhar **durante** a auditoria, mesmo após a verificação inicial.

## 7. Execução — passo a passo
No diretório do pacote:
```bash
chmod +x run.sh linux_auditor.sh
./run.sh /mnt/audit-share inventory.ini
```
Com CVEs importadas e enriquecimento:
```bash
./run.sh /mnt/audit-share inventory.ini \
  advisories.vendor.json kev.json epss.csv
```
**Sintaxe exata:**
```text
./run.sh /mounted/share [inventory.ini] [advisories.json] [kev.json] [epss.csv]
```
Argumentos são posicionais: para usar KEV/EPSS é necessário também fornecer o JSON de avisos. O comando grava `Reports: <destino>` e códigos `Ansible exit` / `Report exit`. Código final zero exige sucesso do Ansible e do processamento dos relatórios; código não zero exige investigação, embora relatórios parciais possam existir.

## 8. Formato do JSON de avisos de fabricante
A correlação utiliza um **array JSON**. Exemplo sintético, nunca utilizar como vulnerabilidade real:
```json
[
  {
    "cve": "CVE-2025-12345",
    "package": "example",
    "manager": "dpkg",
    "distro": "ubuntu",
    "distro_version": "24.04",
    "fixed_version": "1.2.4-1ubuntu1",
    "severity": "HIGH",
    "cvss": 7.5,
    "source": "https://vendor.example/advisory"
  }
]
```
Campos críticos: `cve`, `package`, `manager`, `distro`, `distro_version`, `fixed_version`, `source`; o `severity` e `cvss` devem refletir a publicação do fabricante. Valide manualmente origem, data, arquitetura, epoch, variantes, streams, backports e escopo da distribuição. O projeto **não baixa** esses avisos automaticamente.

**KEV:** informe snapshot JSON compatível com o catálogo CISA, obtido por processo controlado. **EPSS:** informe CSV de snapshot FIRST. Confira a versão do formato antes de uso; formatos externos podem mudar. Registre origem e data dos snapshots.

## 9. Estrutura e leitura dos resultados
```text
/mnt/audit-share/
└── 20261103T140000Z-12345/
    ├── ansible.log
    ├── index.html
    ├── manifest.json
    ├── farm_summary.csv
    ├── farm_security.csv
    ├── farm_cves.csv
    ├── correlated_cves.csv           # se houver avisos
    ├── correlation_summary.json      # se houver avisos
    ├── srv-rhel01/
    │   ├── execution.json
    │   ├── summary.csv
    │   ├── security_checks.csv
    │   ├── packages.csv
    │   └── os-release
    └── srv-ubuntu02/
```
A lista de evidências por host é ilustrativa: outros arquivos podem existir e alguns podem faltar conforme as permissões ou falhas. `index.html` é o painel de consolidação. `manifest.json` contém cobertura de hosts, contagens de severidade das CVEs importadas e status de execução. `farm_security.csv` reúne verificações de segurança. `farm_cves.csv` reúne achados importados; **não** equivale a `correlated_cves.csv`, que é a correlação pacote/aviso.

## 10. CVE, severidade, status e prioridade
| Severidade CVSS v3.x | Faixa |
|---|---|
| CRITICAL | 9.0–10.0 |
| HIGH | 7.0–8.9 |
| MEDIUM | 4.0–6.9 |
| LOW | 0.1–3.9 |
| NONE | 0.0 |
| UNKNOWN | Pontuação ausente/inválida |

A severidade é recebida do aviso; não é inferida apenas do número da versão. A prioridade da implementação é:
- `P1`: `POTENTIALLY_AFFECTED` e CVE presente no KEV **ou** severidade CRITICAL.
- `P2`: `POTENTIALLY_AFFECTED` e severidade HIGH, sem critério P1.
- `REVIEW`: demais casos.

**Estados que exigem interpretação:**
- `POTENTIALLY_AFFECTED`: versão instalada inferior à versão corrigida segundo comparador; **não confirma vulnerabilidade explorável**.
- `SCOPE_MISMATCH`: distribuição/versão do aviso não corresponde ao host.
- `UNVERIFIED_SCOPE`: não foi possível validar escopo do sistema.
- `UNKNOWN_FIXED_VERSION`: aviso sem versão corrigida utilizável.
- `UNKNOWN_COMPARATOR`: comparador nativo indisponível no Controller.
- `NOT_ASSESSED`: não houve cobertura CVE suficiente; **não significa zero vulnerabilidades**.

A presença em KEV indica exploração conhecida catalogada, não invasão deste host. EPSS estima probabilidade de exploração no ecossistema, não a chance individual de comprometimento do servidor.

## 11. Tratamento de falhas e recuperação
| Sintoma | Causa provável | Ação recomendada |
|---|---|---|
| `Refusing to write: share is not a mount point` | Pasta não montada | Conferir NFS/SMB e `mountpoint` |
| `Invalid or unwritable absolute share` | Caminho relativo ou sem escrita | Corrigir montagem e ACL |
| `ansible-playbook not found` | Dependência ausente | Instalar Ansible no Controller |
| `UNREACHABLE_OR_UNREPORTED` | SSH indisponível ou sem evidências | Testar `ansible -m ping`, firewall, chave e logs |
| `FAILED` | Tarefa do playbook falhou | Ler `execution.json` e `ansible.log` |
| `COLLECTOR_WARNING` | Coletor ou inventário retornou erro | Inspecionar logs/evidências locais |
| `UNKNOWN_COMPARATOR` | `dpkg` ou `rpmdev-vercmp` ausente | Instalar comparador adequado no Controller |
| `SCOPE_MISMATCH` | Aviso de outra distribuição/versão | Corrigir escopo do feed |
| HTML/CSV incompleto | Falha de coleta ou share | Conferir `manifest.json`, logs e espaço livre |

**Procedimento de recuperação:** preserve o diretório da execução falha; confirme montagem e SSH; identifique hosts afetados; corrija a causa; execute novamente (novo RUN_ID). Evite editar evidências originais. Caso a sessão SSH tenha caído, procure resíduos temporários no host antes de removê-los manualmente, respeitando a política de mudança.

## 12. Operação segura e boas práticas
- Use uma conta de auditoria de privilégio mínimo; `become: false` é o padrão atual.
- Não armazene senhas ou tokens em `inventory.ini`, logs, argumentos CLI ou relatórios.
- Exija verificação de host keys e segregue acesso de leitura/escrita no NAS.
- Restrinja permissões dos relatórios, registre cadeia de custódia e política de retenção.
- Valide os feeds antes da execução; não use arquivos de exemplo em produção.
- Não interprete um `OK` operacional como comprovação de hardening ou conformidade.
- Para mudança de configuração, use processo separado de change management; este auditor não remedia.

## 13. Mapeamento de frameworks (parcial)
| Referência | Uso operacional |
|---|---|
| NIST CSF 2.0 | Govern, Identify, Protect, Detect; coleta de evidências |
| CIS Controls v8 | Inventário, configuração segura, vulnerabilidades, logs |
| NIST SP 800-53 | CM-8, RA-5, AU-6, SI-2 como referências |
| OWASP | Validação de entradas, tratamento de dados não confiáveis e escape HTML |
| CVSS / CISA KEV / EPSS | Severidade, exploração conhecida e contexto de priorização |
| SLSA | Recomendação de revisão, provenance e controles de supply chain |

Não há evidência de certificação CIS, NIST, SLSA ou avaliação completa de benchmark.

## 14. Testes de aceitação e regressão
**Testes locais fornecidos:**
```bash
bash -n run.sh linux_auditor.sh
python3 -m py_compile consolidate.py correlate.py security_report.py package_inventory.py
python3 -m unittest discover -s tests -v
```
**Teste de integração em laboratório (a executar pelo operador):**
1. Preparar 2 hosts de teste, um Debian/Ubuntu e um RHEL/Rocky, com SSH e Python 3.
2. Montar compartilhamento de teste e confirmar ACLs.
3. Executar sem avisos: verificar relatório e cobertura `NOT_ASSESSED`.
4. Executar com avisos sintéticos isolados: confirmar `SCOPE_MISMATCH`, `UNVERIFIED_SCOPE` e tratamento de CVSS inválido.
5. Derrubar SSH de um host de laboratório e verificar identificação de falha e evidências parciais.
6. Testar compartilhamento indisponível **antes** da execução; verificar recusa.
7. Confirmar limpeza temporária e inexistência de senhas nos relatórios.
8. Registrar duração, volume de dados, códigos de saída e checklist de aprovação.

**Critério de produção:** nenhuma implantação deve depender apenas de testes unitários; faça homologação em rede autorizada com matriz de distribuições e versões suportadas.

## 15. Rotina operacional recomendada
**Antes:** conferir inventário, credenciais, share, feeds e janela autorizada.
**Durante:** acompanhar `ansible.log`, falhas de host e espaço livre no NAS.
**Depois:** validar `manifest.json`, classificar P1/P2/REVIEW, abrir tickets com evidências e responsável, validar achados com avisos do fabricante, arquivar relatórios e aplicar retenção.
**Cadência sugerida:** auditoria diária de saúde e segurança básica; análise CVE após atualização de feeds e em janela operacional aprovada. A programação automática ainda precisa ser configurada externamente.

## 16. Limitações conhecidas e melhorias futuras
- Inventário INI simples para identificação de hosts ausentes; não cobre completamente inventários dinâmicos/YAML.
- Comparação de versões é triagem; backports, streams e variantes exigem validação.
- Fontes KEV/EPSS/advisories são offline, fornecidas pelo operador.
- Algumas verificações podem exigir privilégios adicionais e ferramentas específicas.
- Falha de share durante execução pode impedir relatórios completos.
- `block/rescue/always` não garante limpeza após perda abrupta de conectividade.
- Ainda são recomendados testes reais com NFS/SMB, Molecule, ShellCheck, Ansible Lint e pipeline CI/CD.

## 17. Checklist rápido para o analista
- [ ] Controller com Ansible, Bash, Python 3 e comparadores adequados
- [ ] SSH autenticado e host keys verificadas
- [ ] Grupo `[linux_farm]` preenchido e validado
- [ ] Compartilhamento montado, gravável e com ACL restritiva
- [ ] Feeds confiáveis e datados, quando usados
- [ ] Execução com RUN_ID identificado
- [ ] `manifest.json` revisado e falhas investigadas
- [ ] CVEs validadas contra advisories e backports
- [ ] Relatórios protegidos, retidos e vinculados a tickets

**Fim do manual — Linux Farm Auditor v6.0.**
