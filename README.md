# DioDesafioPython
esse projeto foi criado pela LLM Claude
# Cenários de Detecção — Ransomware e Keylogger

Documento de apoio para simulação segura e detecção no seu lab (Kali + VM vulnerável + Wazuh + Suricata). Todos os cenários usam comportamentos e ferramentas **legítimas do sistema** para simular os padrões que um ransomware/keylogger real deixaria, sem executar malware de verdade. Isso é o mesmo método usado por frameworks de blue team como o **Atomic Red Team** (github.com/redcanaryco/atomic-red-team), que recomendo instalar na sua VM vulnerável para ter uma biblioteca pronta de simulações mapeadas ao MITRE ATT&CK.

---

## Cenário 1 — Ransomware: eliminação de Shadow Copies (T1490)

**O que o atacante faz de verdade:** antes de criptografar, muitos ransomwares tentam impedir a recuperação apagando cópias de sombra e backups locais.

**Simulação seguro no lab (Windows VM):**
```powershell
# Executa o MESMO comando que um ransomware usaria — mas aqui é só para gerar o evento de teste
vssadmin delete shadows /all /quiet
```

**O que você deve ver disparar:**
- Sysmon Event ID 1 (criação de processo) para `vssadmin.exe` com linha de comando contendo `delete shadows`
- Alerta Wazuh correlacionando esse processo como técnica de destruição de backup

**Mapeamento MITRE ATT&CK:** T1490 — Inhibit System Recovery

---

## Cenário 2 — Ransomware: renomeação em massa de arquivos (comportamento de criptografia)

**O que o atacante faz de verdade:** percorre diretórios, lê, criptografa e renomeia centenas/milhares de arquivos em segundos, trocando a extensão.

**Simulação segura no lab (PowerShell, em uma pasta de teste isolada):**
```powershell
# Cria arquivos de teste e simula o padrão de renomeação em massa (sem criptografar nada de verdade)
1..200 | ForEach-Object { "conteudo teste" | Out-File "C:\lab_test\arquivo_$_.txt" }
Get-ChildItem "C:\lab_test\*.txt" | Rename-Item -NewName { $_.Name -replace '\.txt$','.locked' }
```

**O que você deve ver disparar:**
- Alta taxa de eventos de I/O de arquivo em curto intervalo (Sysmon Event ID 11 — FileCreate / renomeação)
- Regra de correlação no Wazuh: "N renomeações no mesmo diretório em X segundos" → alerta de possível criptografia em massa

**Mapeamento MITRE ATT&CK:** T1486 — Data Encrypted for Impact

---

## Cenário 3 — Ransomware: PowerShell ofuscado / living-off-the-land

**Simulação segura:**
```powershell
# Exemplo de flags que ransomware/malware usam para dificultar análise — comando inofensivo aqui
powershell.exe -nop -w hidden -enc <base64 de "Write-Host teste">
```

**O que você deve ver disparar:**
- Sysmon Event ID 1 com flags suspeitas (`-enc`, `-nop`, `-w hidden`)
- Regra Wazuh de "PowerShell com execução oculta/codificada"

**Mapeamento MITRE ATT&CK:** T1059.001 — PowerShell / T1027 — Obfuscated Files or Information

---

## Cenário 4 — Keylogger: registro de hook de teclado

**O que o atacante faz de verdade:** chama `SetWindowsHookEx` com `WH_KEYBOARD_LL` para interceptar teclas globalmente.

**Simulação segura:** em vez de escrever um keylogger funcional, use o **Process Hacker / Process Explorer** na VM para observar processos que registram hooks globais, e crie um processo de teste (ex. um script simples que só abre e fica em memória) para praticar como identificar esse padrão na ferramenta — o objetivo do exercício é treinar a *identificação*, não a criação do mecanismo.

**O que você deve observar/documentar:**
- Processo não assinado digitalmente, rodando sem janela visível
- Hooks de teclado ativos sem motivo (visível via Process Hacker → aba "Handles"/"Hooks" dependendo da versão)
- Entrada de persistência em `HKCU\...\Run` ou `HKLM\...\Run` associada a esse processo

**Mapeamento MITRE ATT&CK:** T1056.001 — Keylogging / T1547.001 — Registry Run Keys

---

## Cenário 5 — Keylogger/Ransomware: beaconing de exfiltração (C2)

**O que o atacante faz de verdade:** envia dados roubados (teclas capturadas, arquivos) em intervalos periódicos e pequenos, para não chamar atenção.

**Simulação segura (gera tráfego de rede de teste no Suricata):**
```bash
# Na VM Kali, gera requisições periódicas pequenas para um domínio de teste (não malicioso)
while true; do curl -s http://exemplo-interno.lab/beacon?id=teste; sleep 60; done
```

**O que você deve ver disparar:**
- Assinatura Suricata de "tráfego periódico de pequeno volume" (beaconing)
- Se usar um domínio .lab inexistente, também testa detecção de DNS incomum

**Mapeamento MITRE ATT&CK:** T1041 — Exfiltration Over C2 Channel


---

## Como documentar cada teste (template de execução)

Para cada cenário executado, preencha:

| Campo | Exemplo |
|---|---|
| Data/hora | 2026-10-28 05:10 |
| Cenário | Cenário 1 — Shadow Copies |
| Comando executado | `vssadmin delete shadows /all /quiet` |
| Técnica MITRE | T1490 |
| Alerta esperado | Wazuh rule ID XXXXX disparado |
| Alerta observado? | Sim/Não |
| Tempo até detecção | X segundos |
| Ação de resposta simulada | Isolar host / revisar regra |
| Observações/ajustes | Regra precisou de afinamento para reduzir falso positivo |
Dá pra estruturar isso como um exercício de analista: entender o mecanismo, mapear os indicadores que ele deixa, e documentar como seria num relatório de incidente. Vou cobrir os dois de forma prática — sem entrar em código funcional de malware (isso eu não produzo), mas com profundidade suficiente pra você reconhecer e analisar casos reais, inclusive no seu mini lab.

## Ransomware — funcionamento prático

**Cadeia de ataque típica (kill chain):**
1. **Acesso inicial** — phishing com anexo malicioso, RDP exposto com credenciais fracas, exploração de vulnerabilidade sem patch, ou compra de acesso já obtido (initial access brokers).
2. **Execução e persistência** — o payload roda, cria chaves de registro (Windows) ou entradas de cron/systemd (Linux) para sobreviver a reboot, e frequentemente desativa ou tenta encerrar backups/Volume Shadow Copies (`vssadmin delete shadows`).
3. **Movimento lateral e escalonamento** — usa credenciais roubadas (Mimikatz, dumps de LSASS) ou ferramentas legítimas já no ambiente ("living off the land": PsExec, WMI, PowerShell) para se espalhar pela rede antes de disparar a criptografia.
4. **Exfiltração (dupla extorsão)** — ransomware moderno copia dados sensíveis para servidores do atacante *antes* de criptografar, para ameaçar vazamento mesmo que a vítima tenha backup.
5. **Criptografia** — usa cifra simétrica (AES) para os arquivos e cifra assimétrica (RSA/ECC) para proteger a chave simétrica, que só o atacante consegue decifrar. Extensões de arquivos são trocadas, um arquivo de resgate (`README.txt`, `DECRYPT_INSTRUCTIONS.html`) é deixado em cada pasta.
6. **Extorsão** — nota de resgate com instruções de pagamento (geralmente cripto) e prazo.

**Indicadores de comprometimento (IOCs) a observar:**
- Picos de leitura/escrita em disco e renomeação em massa de arquivos em curto intervalo
- Processos incomuns chamando `vssadmin`, `wbadmin`, `bcdedit` (sinal de que estão apagando recuperação)
- Tráfego de saída anômalo e volumoso antes do evento de criptografia (fase de exfiltração)
- Uso de PowerShell com flags de ofuscação (`-enc`, `-nop`, `-w hidden`)
- Criação de contas administrativas novas ou uso de ferramentas como PsExec fora do padrão

## Keylogger — funcionamento prático

**Tipos principais:**
- **Baseado em hook de API** (mais comum): usa chamadas do sistema para interceptar eventos de teclado — no Windows, `SetWindowsHookEx` com `WH_KEYBOARD_LL`; em Linux, acesso a `/dev/input/eventX` ou hooks em X11/Wayland.
- **Baseado em kernel/driver**: intercepta no nível do driver de teclado, mais difícil de detectar mas exige privilégios elevados.
- **Baseado em formulário/navegador**: injeta JavaScript malicioso para capturar campos antes do envio (comum em ataques web).
- **Hardware**: dispositivo físico entre teclado e porta USB — não deixa rastro em software, só detecção visual/física.

**Comportamento típico:**
1. Captura teclas e frequentemente também clipboard, capturas de tela periódicas e janela ativa (para contextualizar o que foi digitado — ex. "usuário digitou senha na janela do banco X").
2. Grava localmente em arquivo oculto/criptografado.
3. Exfiltra os dados — por e-mail, upload para servidor C2, ou via protocolos legítimos (DNS tunneling, HTTPS para parecer tráfego normal) em intervalos programados.
4. Busca persistência via chaves de registro `Run`, tarefas agendadas, ou serviços disfarçados de processos legítimos.

**Indicadores a observar:**
- Processos não assinados ou com nome similar a processos do sistema (`scvhost.exe` em vez de `svchost.exe`)
- Hooks de teclado registrados por processos sem motivo aparente (ferramentas como Process Explorer / Process Hacker mostram isso)
- Conexões de rede periódicas e pequenas (batendo em intervalo fixo) para domínios desconhecidos
- Entradas de inicialização automática não reconhecidas no registro ou nas tarefas agendadas

## Como identificar na prática (ligado ao seu lab)

Já que você está montando Wazuh + Suricata, esse é exatamente o ambiente certo para praticar detecção real:
- **Sysmon + Wazuh**: configure regras para alertar em criação de processos suspeitos (`vssadmin`, `wmic shadowcopy delete`), execução de PowerShell ofuscado, e hooks de teclado incomuns.
- **Suricata**: crie regras para detectar beaconing (tráfego periódico pequeno) e exfiltração em volume.
- **YARA**: pode escrever regras de assinatura para identificar famílias conhecidas de ransomware/keylogger em amostras (ex. em repositórios como o MalwareBazaar, para análise estática segura em ambiente isolado).

## Como documentar (formato de relatório de incidente)

Um bom relatório de SOC geralmente segue essa estrutura:

1. **Resumo executivo** — o que aconteceu, impacto, status
2. **Linha do tempo** — timestamps de cada fase (acesso inicial → persistência → movimento lateral → exfiltração → impacto)
3. **IOCs coletados** — hashes de arquivo, IPs/domínios de C2, chaves de registro, nomes de processo
4. **Mapeamento MITRE ATT&CK** — associar cada etapa a uma técnica (ex: T1486 Data Encrypted for Impact, T1056.001 Keylogging, T1059.001 PowerShell)
5. **Ações de contenção e erradicação** — isolamento de host, revogação de credenciais, remoção de persistência
6. **Lições aprendidas e recomendações** — patches faltando, controles que falharam, melhorias propostas

Se quiser, posso te ajudar a montar esse template de relatório como um documento reutilizável, ou criar cenários de detecção específicos (regras Sysmon/Wazuh/Suricata) para você testar essas duas ameaças no seu lab.

Isso vira a base de um portfólio de testes de detecção — muito valioso para entrevistas de SOC júnior, porque mostra que você não só configurou as ferramentas, mas validou que elas funcionam.
