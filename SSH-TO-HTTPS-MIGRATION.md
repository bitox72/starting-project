# Migrazione da SSH a HTTPS per autenticazione GitHub

## Obiettivo

Consentire sia all'utente (Git Bash) che a Claude (Bash tool) di eseguire `git push` verso GitHub senza dover inserire una passphrase ad ogni operazione.

---

## Contesto iniziale

Il punto di partenza era un sistema **senza alcuna chiave SSH configurata**. I passi eseguiti per arrivare alla configurazione SSH:

1. Generazione della chiave ED25519:
   ```bash
   ssh-keygen -t ed25519 -C "stefano.bitossi@gmail.com"
   ```
2. Aggiunta della chiave pubblica su GitHub (Settings → SSH and GPG keys)
3. Configurazione del remote del repository in formato SSH:
   ```bash
   git remote set-url origin git@github.com:bitox72/starting-project.git
   ```

**Problema emerso:** ad ogni `git push` veniva richiesta la passphrase della chiave ED25519, sia nelle sessioni Git Bash dell'utente che nel Bash tool di Claude.

- Sistema: Windows 11, Git Bash (MSYS2), shell di Claude = Git Bash isolato

---

## Tentativi falliti

### 1. Script ssh-agent in `~/.bash_profile`

**Approccio:** aggiungere uno script che avvia `ssh-agent` automaticamente e aggiunge la chiave SSH alla prima apertura di Git Bash.

```bash
# ~/.bash_profile
env=~/.ssh/agent.env
agent_load_env () { test -f "$env" && . "$env" >| /dev/null ; }
agent_start () {
    (umask 077; ssh-agent >| "$env")
    . "$env" >| /dev/null ;
}
agent_load_env
if ! ssh-add -l >| /dev/null 2>&1; then
    agent_start
    ssh-add ~/.ssh/id_ed25519
fi
```

**Perché fallisce:** risolve il problema per le sessioni Git Bash dell'utente (chiede la passphrase una volta sola), ma il Bash tool di Claude avvia una shell separata e isolata per ogni comando. Quella shell non eredita l'`ssh-agent` dell'utente, quindi non ha accesso alle chiavi già caricate e la passphrase viene richiesta ad ogni esecuzione.

---

### 2. Windows OpenSSH Authentication Agent + `core.sshCommand`

**Approccio:** usare il servizio Windows OpenSSH Agent (persistente tra le sessioni) e configurare git per usare `C:/Windows/System32/OpenSSH/ssh.exe`.

```bash
# Verifica servizio Windows (PowerShell)
Get-Service ssh-agent  # Status: Running, StartType: Automatic

# Configura git per usare Windows ssh
git config --global core.sshCommand "C:/Windows/System32/OpenSSH/ssh.exe"
```

**Perché fallisce:** quando `C:/Windows/System32/OpenSSH/ssh.exe` viene invocato dall'interno di MSYS2 (Git Bash), non riesce a connettersi al Windows OpenSSH Agent tramite la named pipe `\\.\pipe\openssh-ssh-agent`. Il binario Windows nativo non può comunicare con il layer POSIX di MSYS2 per accedere al socket dell'agent. Risultato: il push rimaneva appeso senza risposta.

---

### 3. `SSH_AUTH_SOCK` puntato alla named pipe Windows

**Approccio:** impostare la variabile d'ambiente `SSH_AUTH_SOCK` in `~/.bash_profile` in modo che Git Bash usi il Windows OpenSSH Agent.

```bash
# Primo tentativo (percorso errato)
export SSH_AUTH_SOCK=//.pipe/openssh-ssh-agent

# Secondo tentativo (percorso corretto per MSYS2)
export SSH_AUTH_SOCK=//./pipe/openssh-ssh-agent
```

**Perché fallisce:** Git Bash (MSYS2) non supporta named pipe di Windows tramite la sintassi POSIX. Il risultato di `ssh-add -l` era:
```
Error connecting to agent: No such file or directory
```
MSYS2 non traduce `//./pipe/...` in `\\.\pipe\...` come farebbe un processo Windows nativo.

---

### 4. `credential.helper=manager` in conflitto

**Approccio:** dopo il passaggio a HTTPS, configurare `credential.helper=store` con percorso assoluto per evitare dipendenze da `HOME` (vuoto nel Bash tool di Claude).

```bash
git config --global credential.helper "store --file C:/Users/stefano/.git-credentials"
```

**Perché fallisce parzialmente:** il sistema aveva già `credential.helper=manager` a livello di configurazione di sistema (Git for Windows imposta Git Credential Manager per default). Con due helper configurati, git li prova in ordine: `manager` veniva invocato per primo, aprendo una finestra GUI di autenticazione sul desktop dell'utente.

Verifica del conflitto:
```bash
git config --list | grep credential
# credential.helper=manager         ← sistema (Git for Windows)
# credential.helper=store --file C:/Users/stefano/.git-credentials  ← globale
```

Tentativo di override con stringa vuota:
```bash
git config --global credential.helper ""
git config --global --add credential.helper "store --file C:/Users/stefano/.git-credentials"
```
Parzialmente efficace per il Bash tool di Claude (HOME vuoto = il manager non trovava credenziali e passava allo store), ma non abbastanza per la sessione dell'utente.

---

### 5. Script ssh-agent in `~/.bashrc`

**Problema scoperto:** durante la diagnosi è emerso che `~/.bashrc` conteneva:

```bash
# SSH Agent - avvia automaticamente e carica la chiave
eval "$(ssh-agent -s)" > /dev/null
ssh-add ~/.ssh/id_ed25519 2>/dev/null
```

Questo causava una richiesta di passphrase **all'avvio di ogni shell**, inclusa ogni invocazione del Bash tool di Claude. Era la causa principale dei prompt ripetuti anche dopo il passaggio a HTTPS.

---

## Soluzione finale

### Passaggi eseguiti

**1. Cambiare il remote da SSH a HTTPS**

```bash
git remote set-url origin https://github.com/bitox72/starting-project.git
```

**2. Creare un Fine-Grained Personal Access Token su GitHub**

Percorso: GitHub → Settings → Developer settings → Personal access tokens → Fine-grained tokens → Generate new token

Permessi assegnati:
- Repository: All repositories
- Contents: Read and Write
- Issues: Read and Write
- Pull requests: Read and Write

**3. Salvare le credenziali nel file store**

```bash
git credential-store --file ~/.git-credentials store <<EOF
protocol=https
host=github.com
username=bitox72
password=<TOKEN>
EOF
```

**4. Configurare credential.helper globale**

```bash
git config --global credential.helper store
```

**5. Impostare il percorso assoluto (necessario per il Bash tool di Claude)**

Il Bash tool di Claude ha `HOME` vuoto, quindi `~` non si risolve correttamente:

```bash
git config --global credential.helper "store --file C:/Users/stefano/.git-credentials"
```

**6. Override del credential.helper a livello locale del repo**

Per garantire che il `manager` di sistema non venga mai invocato nel repo di progetto:

```bash
git config --local credential.helper "store --file C:/Users/stefano/.git-credentials"
```

**7. Rimuovere lo script ssh-agent da `~/.bashrc`**

```bash
# Rimosso da ~/.bashrc:
# eval "$(ssh-agent -s)" > /dev/null
# ssh-add ~/.ssh/id_ed25519 2>/dev/null
```

**8. Rimuovere core.sshCommand (non più necessario con HTTPS)**

```bash
git config --global --unset core.sshcommand
```

---

## Perché la soluzione finale funziona

1. **Remote HTTPS** — git usa il trasporto HTTP invece di SSH, eliminando completamente la necessità di chiavi SSH e del relativo agent.

2. **Credential store con percorso assoluto** — il file `~/.git-credentials` contiene il token in chiaro ma protetto dai permessi del filesystem. Il percorso assoluto (`C:/Users/stefano/...`) funziona sia quando `HOME` è impostato (sessione utente) sia quando è vuoto (Bash tool di Claude).

3. **Local config come override** — la configurazione a livello di repo (`.git/config`) ha precedenza su quella globale e di sistema. Questo garantisce che `credential.helper=manager` (impostato da Git for Windows a livello di sistema) non venga mai invocato per questo repository, evitando la finestra GUI.

4. **Rimozione ssh-agent da `~/.bashrc`** — ogni nuova shell (incluse quelle del Bash tool di Claude) non avvia più un agent SSH né richiede la passphrase della chiave.

---

## Stato finale dei file modificati

| File | Modifica |
|------|----------|
| `~/.bash_profile` | Rimosso script ssh-agent (sostituito e poi svuotato) |
| `~/.bashrc` | Rimosso `eval ssh-agent` e `ssh-add` |
| `~/.git-credentials` | Aggiunto token HTTPS per github.com |
| `.git/config` (locale) | Aggiunto `credential.helper=store --file ...` |
| Git global config | `credential.helper` con percorso assoluto, rimosso `core.sshCommand` |
