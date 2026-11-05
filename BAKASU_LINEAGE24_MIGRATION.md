# Migração: ReSukiSU ➜ BakaSU e LineageOS 23.2 ➜ 24.0

Kernel: Samsung Exynos 9830 (`universal9830`, Linux 4.19.325, non-GKI)
Dispositivos: Galaxy S20 / S20+ / S20 Ultra / Note20 / Note20 Ultra (x1s, y2s, z3s, c1s, c2s e variantes LTE) e r8s
Branch: `claude/beautiful-turing-uvuzcl`
Data: 2026-11-05

---

## 1. Resumo

| Item | Antes | Depois |
|---|---|---|
| Base do kernel | LineageOS `lineage-23.2` (`0d6cd86ea`) | LineageOS `lineage-24.0` (`97369af7b`) |
| Root (submódulo `KernelSU/`) | `ReSukiSU/ReSukiSU` @ `6d18926a` | `Baka-SU/BakaSU` @ `516d3e87` (`v4.2.0-rc3-86`, `main`) |
| Método de hook | `CONFIG_KSU_MANUAL_HOOK` | `CONFIG_KSU_MANUAL_HOOK` (sem mudança) |
| Hook `newfstat` | no `COMPAT_SYSCALL_DEFINE2(newfstat)` (errado) | no `SYSCALL_DEFINE2(newfstat)` nativo (corrigido) |
| `localversion-st` | `-ReSukiSU` | `-BakaSU` |

**Resultado da verificação:** com `exynos9830_defconfig + c2s.config + ksu.config`, todas as checagens de hook do BakaSU passam, e `fs/*.c`, `kernel/reboot.c`, `security/selinux/` e `drivers/kernelsu/` compilam sem erros nem warnings (veja a seção 5).

---

## 2. ReSukiSU ➜ BakaSU

### 2.1 O que é o BakaSU

O BakaSU (<https://bakasu.org>, código em <https://github.com/Baka-SU/BakaSU>) é **o próprio ReSukiSU com outro nome**. O README do projeto diz *"Formerly known as: ReSukiSU"*, e a troca foi feita no commit `97c102ba` *"treewide: rename project (#461)"*. O commit do ReSukiSU que o projeto usava (`6d18926a`) está no histórico do BakaSU. Por isso a migração é uma atualização do submódulo, sem trocar de projeto: são 99 commits à frente.

Mudanças relevantes do rename para quem usa o kernel:

- O namespace interno do manager passou de `com.resukisu.resukisu` para `org.bakasu.bakasu`. O **`applicationId` padrão continua `com.resukisu.resukisu`** (`manager/build.gradle.kts`), então o manager atualiza por cima do antigo.
- O suporte multi-manager para MKSU / RKSU / SukiSU-Ultra **foi removido**. Os managers aceitos agora são **KernelSU oficial, KOWSU e BakaSU** (o build mostra isso: `Supported Managers: KernelSU Official, KOWSU, BakaSU`).
- O `Kbuild` agora mostra `REPO_NAME := BakaSU`, e a versão no manager aparece como `v4.2.0-rc3-516d3e87`. O version code é `35257`.

### 2.2 Alterações feitas

**`.gitmodules`**: URL do submódulo trocada e branch fixada:

```ini
[submodule "KernelSU"]
	path = KernelSU
	url = https://github.com/Baka-SU/BakaSU.git
	branch = main
```

**`KernelSU`** (gitlink): aponta para `516d3e87fbc986ccefdde2c2765f1700612441a9`, o último commit da `main` do BakaSU.

O symlink `drivers/kernelsu -> ../KernelSU/kernel` e as entradas em `drivers/Makefile` e `drivers/Kconfig` continuam iguais. São exatamente o que o `kernel/setup.sh` do BakaSU gera.

**`arch/arm64/configs/ksu.config`**: reescrito:

- Removido `CONFIG_KSU_ALLOWLIST_WORKAROUND=y`. **Essa opção não existe** no Kconfig do BakaSU (nem existia no ReSukiSU recente) e era ignorada em silêncio.
- Definidas de forma explícita as opções do BakaSU para hooks manuais em kernel < 6.8: `CONFIG_KSU_MANUAL_HOOK_AUTO_SETUID_HOOK`, `..._AUTO_INITRC_HOOK` e `..._AUTO_INPUT_HOOK` (todas `=y`). Com elas, `kernel/sys.c`, `fs/read_write.c` e `drivers/input/input.c` **não** precisam de patch.
- Definido de forma explícita `CONFIG_KSU_MULTI_MANAGER_SUPPORT=y`.
- Definido de forma explícita `CONFIG_KALLSYMS_ALL=y`, que já vinha no `exynos9830_defconfig`. Com ela o BakaSU dispensa os "static symbol exports" de `security/selinux/ss/services.c` (`security_dump_masked_av`, `context_struct_compute_av`). Veja a seção 4.4.

**`localversion-st`**: `-ReSukiSU` ➜ `-BakaSU`. O `uname -r` fica `4.19.325-ivnls-BakaSU…`.

**`arch/arm64/configs/defconfig`**: removida a linha `CONFIG_LOCALVERSION="-sm8250-ivnls-ReSukiSU"`. Esse é o defconfig **genérico** de arm64 (não é usado no 9830), e o texto `sm8250` (Snapdragon 865) não tem relação com este kernel. O arquivo voltou a ser idêntico ao upstream.

### 2.3 Como atualizar o BakaSU no futuro

```bash
git submodule update --init --remote KernelSU
```

```bash
git add KernelSU && git commit -m "chore: bump BakaSU"
```

> ⚠️ O `Kbuild` do BakaSU **exige** que `KernelSU/` seja um checkout git (submódulo). Se não for, o build para com *"You should use BakaSU as a git submodule"*. Se o clone for shallow, ele roda `git fetch --unshallow` durante o build, porque o version code vem de `git rev-list --count`. Em CI, clone com `--recurse-submodules` e sem `--depth`. Se o CI não tiver rede durante o build, o version code sai errado e o manager pode recusar o kernel.

---

## 3. LineageOS 23.2 ➜ 24.0

### 3.1 Origem

Upstream: <https://github.com/LineageOS/android_kernel_samsung_universal9830>. O branch `lineage-24.0` (Android 17) já existe e está em `97369af7b` ("arm64: configs: Enable CONFIG_SECURITYFS", 2026-10-12).

O branch `lineage-24.0` foi **refeito (rebase)** pela LineageOS a partir do `lineage-23.2`. Os 8 commits de dispositivo da 23.2 (uwb, decon c2s, wacom, vrr_lfd, painel 60hs, input) aparecem nele com hashes novos e **conteúdo idêntico**.

### 3.2 Como foi integrado

Por **merge** (`50528d727`), sem rebase. Assim o histórico remoto não precisa ser reescrito (não há force-push). O merge não teve conflitos. Depois dele, a diferença entre este repositório e o `lineage-24.0` puro é **só** a integração KSU/BakaSU:

```
.gitignore, .gitmodules, KernelSU, arch/arm64/configs/ksu.config,
drivers/Kconfig, drivers/Makefile, drivers/kernelsu, fs/exec.c, fs/open.c,
fs/stat.c, kernel/reboot.c, localversion-cip, localversion-st,
security/selinux/selinuxfs.c
```

### 3.3 O que a 24.0 traz (30 commits novos)

- `clone3()` (syscall nova) com `CLONE_CLEAR_SIGHAND`, `set_tid`, `CLONE_PIDFD` fix e `copy_thread_tls` para arm64. O Android 17 (bionic/ART) usa `clone3`.
- `CAP_CHECKPOINT_RESTORE` e as mudanças ligadas a ela em prctl, `/proc/*/map_files`, `ns_last_pid` e `TIOCSLCKTRMIOS`.
- BPF JIT arm64: JMP32, `bpf_line_info`, sequência movn/movk e fix de endereço de subprog.
- SELinux: `genfscon` para `securityfs`, mais `CONFIG_SECURITYFS=y` no `exynos9830_defconfig`.

Nenhum desses commits mexe nos arquivos dos hooks (`fs/exec.c`, `fs/stat.c`, `fs/open.c`, `kernel/reboot.c`, `security/selinux/selinuxfs.c`). Por isso os hooks foram mantidos sem ajuste de contexto.

### 3.4 Para usar na árvore LineageOS 24

No device tree / `BoardConfig` do LineageOS, aponte o kernel para este repositório no branch atual e inclua o fragmento KSU, por exemplo:

```make
TARGET_KERNEL_CONFIG := exynos9830_defconfig c2s.config ksu.config
```

Troque `c2s.config` pelo fragmento do seu aparelho: `x1s`, `y2s`, `z3s`, `c1s`, `r8s` ou as variantes `*lte`.

---

## 4. Revisão da implementação (hooks manuais)

A implementação foi comparada com a referência oficial <https://bakasu.org/guide/manual-integrate.html> (repositório `Baka-SU/Baka-SU.github.io`, `docs/guide/manual-integrate.md`) e com o código do BakaSU (`kernel/tools/manual_hook_check.mk`, `static_export_check.mk`, `feature/sucompat.c` e `runtime/ksud_integration.c`).

### 4.1 ❌ ➜ ✅ Corrigido: `ksu_handle_newfstat_ret` estava no syscall errado

**Problema.** O commit `1595330fc` colocou o hook em `COMPAT_SYSCALL_DEFINE2(newfstat, …, struct compat_stat __user *statbuf)`, que é o syscall dos processos **32-bit**, e fez cast de `compat_stat **` para `stat **` para calar o warning. O `SYSCALL_DEFINE2(newfstat)` **nativo (64-bit)**, que a documentação manda hookar, ficou sem hook.

**Por que importa.** O `ksu_handle_newfstat_ret` serve para o `init` (64-bit no arm64) ver o `st_size` do `init.rc` já somado ao tamanho do rc injetado pelo KernelSU. Com o hook só no compat:

1. o `fstat()` do `init` 64-bit **nunca** passava pelo hook, e o tamanho do `init.rc` não era ajustado. Isso pode truncar a leitura do rc injetado (serviços do `ksud`, como o `post-fs-data` e o `boot-completed` dos módulos);
2. se um processo 32-bit chamasse o hook, o handler escreveria um `long` de 8 bytes em `offsetof(struct stat, st_size)` (offset 48 no layout arm64) dentro de um `struct compat_stat` (layout ARM 32-bit, `st_size` em outro offset e com 4 bytes). Isso corromperia memória do usuário. Na prática só acontece se o chamador for `init` e o fd for o `init.rc`, mas o código estava errado.

A checagem de build do BakaSU **não detecta** esse erro, porque ela só faz `grep` do nome da função em `fs/stat.c`.

**Correção** (`fs/stat.c`):

```diff
 SYSCALL_DEFINE2(newfstat, unsigned int, fd, struct stat __user *, statbuf)
 {
 	struct kstat stat;
 	int error = vfs_fstat(fd, &stat);

 	if (!error)
 		error = cp_new_stat(&stat, statbuf);

+#ifdef CONFIG_KSU_MANUAL_HOOK
+	ksu_handle_newfstat_ret(&fd, &statbuf);
+#endif
 	return error;
 }
 …
 COMPAT_SYSCALL_DEFINE2(newfstat, unsigned int, fd,
 		       struct compat_stat __user *, statbuf)
 {
 …
 		error = cp_compat_stat(&stat, statbuf);
-
-	#ifdef CONFIG_KSU_MANUAL_HOOK
-		// Cast to avoid -Wincompatible-pointer-types
-		ksu_handle_newfstat_ret(&fd, (struct stat __user **)&statbuf);
-	#endif
-
 	return error;
 }
```

Os processos 32-bit continuam cobertos pelo `ksu_handle_fstat64_ret` em `SYSCALL_DEFINE2(fstat64)` (arm64 define `__ARCH_WANT_COMPAT_STAT64`), que já estava correto. O 32-bit é o caso que a documentação marca como opcional.

### 4.2 Estado de cada hook

| Hook (doc BakaSU) | Arquivo / função | Status |
|---|---|---|
| `ksu_handle_stat` | `fs/stat.c` → `newfstatat`, `fstatat64` | ✅ OK |
| `ksu_handle_newfstat_ret` | `fs/stat.c` → `newfstat` | ✅ **Corrigido** (4.1) |
| `ksu_handle_fstat64_ret` | `fs/stat.c` → `fstat64` | ✅ OK |
| `ksu_handle_execveat` / `ksu_handle_post_execveat` | `fs/exec.c` → `do_execveat_common` | ✅ OK, veja a observação 4.3 |
| `ksu_handle_faccessat` | `fs/open.c` → `SYSCALL faccessat` (padrão 4.19+) | ✅ OK |
| `ksu_handle_sys_reboot` | `kernel/reboot.c` → `SYSCALL reboot` | ✅ OK |
| `ksu_handle_setresuid` | — | ✅ Não precisa (`AUTO_SETUID_HOOK=y`, kernel < 6.8) |
| `ksu_handle_sys_read` | — | ✅ Não precisa (`AUTO_INITRC_HOOK=y`, kernel < 6.8) |
| `ksu_handle_input_handle_event` | — | ✅ Não precisa (`AUTO_INPUT_HOOK=y`) |
| `write_op`, `sel_handle_status_ops` sem `static` | `security/selinux/selinuxfs.c` | ✅ OK |
| `security_dump_masked_av`, `context_struct_compute_av` sem `static` | `security/selinux/ss/services.c` | ⚠️ Não feito, e não precisa enquanto `KALLSYMS_ALL=y` (4.4) |
| Hooks incompatíveis (`ksu_vfs_read_hook`, `is_ksu_transition`, `ksu_handle_rename`) | — | ✅ Ausentes, como deve ser |

### 4.3 ⚠️ Observação: `fs/exec.c`, o post-hook recebe um `filename` já liberado

O hook atual envolve a chamada em `do_execveat_common()`:

```c
ksu_handle_execveat(&fd, &filename, &argv, &envp, &flags);
retval = __do_execve_file(fd, filename, argv, envp, flags, NULL);
ksu_handle_post_execveat(&fd, &filename, &argv, &envp, &flags, &retval);
```

Na 4.19, o `__do_execve_file()` chama `putname(filename)` antes de retornar, tanto no sucesso quanto no erro. Então o `ksu_handle_post_execveat` recebe um ponteiro **já liberado**.

- **Hoje é seguro.** A versão atual do BakaSU (`feature/sucompat.c`) só faz `IS_ERR(filename)` e calcula `&filename->name` (aritmética de ponteiro, sem ler memória). Depois chama `ksu_handle_post_execve_common()`, que nem usa `filename`.
- **Risco futuro.** Se uma versão nova do BakaSU passar a ler `filename->name` no post-hook (o código de `preinit/` já faz isso, mas hoje não é compilado com manual hook), vira use-after-free.
- **Por que não troquei.** O padrão "3.14+" da documentação põe o post-hook em `out_ret:` antes do `putname`. Na 4.19 isso só cobre o **caminho de erro**, porque o sucesso tem `putname` + `return` separados. O padrão atual chama o post-hook em todos os caminhos e cobre `execve`, `execveat` e as versões compat. **Ele não é o padrão "deprecated" (`do_execve`)**, que a doc desaconselha no Android 17 QPR2+. Por isso mantive o padrão atual.
- **Ação recomendada.** Ao atualizar o BakaSU, confira se `ksu_handle_post_execveat` continua sem desreferenciar `filename`:

```bash
grep -n -A12 "int ksu_handle_post_execveat" KernelSU/kernel/feature/sucompat.c
```

### 4.4 ⚠️ Dependência de `CONFIG_KALLSYMS_ALL`

O BakaSU novo passou a exigir também que `security_dump_masked_av` e `context_struct_compute_av` (em `security/selinux/ss/services.c`) não sejam `static`, **a menos que** `CONFIG_KALLSYMS_ALL=y`. Este kernel tem `KALLSYMS_ALL=y` no `exynos9830_defconfig`, e agora também no `ksu.config`, então a checagem é pulada e o BakaSU resolve os símbolos via kallsyms. **Se alguém desligar o `KALLSYMS_ALL`** (para reduzir o tamanho, por exemplo), o build vai falhar e será preciso tirar o `static` dessas duas funções, como manda a documentação.

### 4.5 Outros pontos (sem impacto funcional, não alterados)

- **`build_kernel.sh`** é o script original da Samsung e está quebrado: usa `make exynos9830-x1slte_defconfig`, que **não existe** (o certo é `exynos9830_defconfig x1slte.config`), não define `ARCH`, `CC`, `LLVM` nem `O=`, e não inclui o `ksu.config`. Exemplo de comando que funciona:

```bash
make O=out ARCH=arm64 LLVM=1 CROSS_COMPILE=aarch64-linux-gnu- exynos9830_defconfig c2s.config ksu.config
```

```bash
make O=out ARCH=arm64 LLVM=1 CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc)
```

- **Cabeçalho do `ksu.config`.** Dizia "ExtremeKernel config file" (sobra de outro projeto). Corrigido.
- **Estilo.** As declarações `extern` dos hooks estão indentadas com tab no escopo de arquivo (`fs/stat.c`, `fs/open.c`, `fs/exec.c`, `kernel/reboot.c`) e há espaços no fim de algumas linhas (`__attribute__((hot)) `). É só cosmético e não mexi, para não gerar diff sem necessidade.
- **`faccessat` em 32-bit.** O syscall `access()` (que só existe em compat/32-bit) chama `do_faccessat` direto e não passa pelo hook. Isso bate com a documentação (padrão 4.19+). Só afeta `su` 32-bit chamado via `access()`.
- **SUSFS.** Não está integrado. Se for adicionar, troque `CONFIG_KSU_MANUAL_HOOK` por `CONFIG_KSU_SUSFS` (são opções exclusivas da mesma `choice`) e aplique o patch do susfs4ksu para 4.19. O BakaSU só suporta SUSFS em 4.3+ e o lado kernel fica por sua conta.

---

## 5. Verificação

Ambiente: Ubuntu, clang 18.1.3 + ld.lld (LLVM=1). O CI e a LineageOS usam o clang da AOSP, então diferenças de warning são esperadas.

```bash
make O=out ARCH=arm64 LLVM=1 CROSS_COMPILE=aarch64-linux-gnu- exynos9830_defconfig c2s.config ksu.config
```

`.config` resultante:

```
CONFIG_KSU=y
CONFIG_KSU_MANUAL_HOOK=y
# CONFIG_KSU_TRACEPOINT_HOOK is not set
# CONFIG_KSU_SUSFS is not set
CONFIG_KSU_MANUAL_HOOK_AUTO_SETUID_HOOK=y
CONFIG_KSU_MANUAL_HOOK_AUTO_INITRC_HOOK=y
CONFIG_KSU_MANUAL_HOOK_AUTO_INPUT_HOOK=y
CONFIG_KSU_MULTI_MANAGER_SUPPORT=y
CONFIG_KALLSYMS_ALL=y
CONFIG_SECURITYFS=y          # vindo da lineage-24.0
CONFIG_LTO_CLANG=y
```

Checagens do BakaSU durante o build (`drivers/kernelsu/`):

```
-- BakaSU version code: 35257
-- BakaSU version name: v4.2.0-rc3-516d3e87
-- KERNEL_VERSION: 4.19
-- KERNEL_TYPE: Non-GKI
-- BakaSU: using Manual Hook
-- BakaSU/manual_hook: You are using LSM hooks for setuid hooks.
-- BakaSU/manual_hook: You are using LSM hooks for init rc hooks.
-- BakaSU/manual_hook: You are using input_handler for input hooks.
-- BakaSU/manual_hook: ksu_handle_execveat found
-- BakaSU/manual_hook: ksu_handle_faccessat found
-- BakaSU/manual_hook: ksu_handle_stat found
-- BakaSU/manual_hook: ksu_handle_newfstat_ret found
-- BakaSU/manual_hook: ksu_handle_fstat64_ret found
-- BakaSU/manual_hook: ksu_handle_sys_reboot found
```

Compilação de `fs/stat.o`, `fs/exec.o`, `fs/open.o`, `kernel/reboot.o`, `security/selinux/` e `drivers/kernelsu/`: **sem erros e sem warnings**.

**Build completo (`Image`, com LTO):** _em andamento; o resultado será adicionado aqui._

**Ainda falta testar no aparelho.** A versão anterior (ReSukiSU) foi validada pelo autor no SM-N986B (c2s) com Lineage 23.2. Esta combinação (BakaSU + Lineage 24.0) ainda precisa de: boot, `su` pelo manager, `ksud` subindo (módulos / `post-fs-data`) e o `init.rc` injetado (é o que a correção 4.1 afeta).

---

## 6. Commits

1. `Merge LineageOS lineage-24.0 into ReSukiSU/BakaSU tree`: base 23.2 ➜ 24.0.
2. `feat: migrate ReSukiSU to BakaSU and fix newfstat manual hook`: submódulo, `ksu.config`, `fs/stat.c`, `localversion-st`, `defconfig` e este documento.
