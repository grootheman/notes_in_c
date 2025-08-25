C 語言字串／字元處理安全全攻略（Fedora/Linux 與 Linux kernel 取向）

這份筆記把先前談到的重點全部系統化：字串本質 → 常見 API 的安全/危險點 → kernel 推薦替代 → Fedora/Linux 實務。每個 API 都有「⚠️ 何時危險」「✅ 正確用法」與短範例。

0) 心智模型：為什麼 C 的字串容易出事？

C 沒有內建 string 類型；「字串」= 以 '\0' 結尾的 char 陣列（或指標）。

幾乎所有字串 API 都假設「你會負責保證緩衝區（buffer）足夠大、而且有結尾 \0」。

長度（位元組）≠ 文字字元數：UTF-8 一個字元可能佔 1～4 個位元組，strlen() 回傳的是位元組數，不是人眼看到的「字數」。

動態配置與釋放全靠你（malloc/free）。

跨執行緒與重入性：像 strtok() 會修改原字串且使用靜態狀態，不可同時在多執行緒或巢狀使用，改用 strtok_r()/strsep()。

1) 「先選工具」決策樹（User space 首選）

格式化輸出到固定緩衝區 → 用 snprintf()（一定給 size）。

拷貝字串到固定緩衝區 → 用 strlcpy()（glibc 2.38+ 已提供；舊環境可用 libbsd），或自己用 snprintf(dst, size, "%s", src)。 
GitHub
+1
isopenbsdsecu.re

拼接到固定緩衝區 → 用 strlcat()（或 snprintf 重新格式化整段）。 
GitHub
isopenbsdsecu.re

來源大小未知（讀一行） → 用 getline()（GNU/POSIX；需要 _GNU_SOURCE 或適當 feature test）。

需要剛好大小的拷貝 → 用 strdup()（別忘了 free）。

要分詞（tokenize） → 優先 strtok_r() 或 strsep()；避免 strtok()。

計算長度但怕沒終止符 → 用 strnlen(ptr, max)，別裸用 strlen()。

Kernel space：盡量用 strscpy()（永遠 NUL 結尾、截斷回傳負值）與 scnprintf()（回傳實際寫入長度），Kernel 文件也明言 不要新增 strcpy() 用例。 
infradead.org
LWN.net
Debian Manpages

2) API-by-API：危險情境 & 正確用法
strcpy(dst, src)（避免）

危險：不檢查長度 → 一旦 src 長於 dst 直接溢位。

安全替代：strlcpy() / snprintf() /（kernel）strscpy()。

示例

// ⚠️ 可能溢位
char buf[8];
strcpy(buf, "abcdefghij"); // 爆

// ✅ 安全（截斷）
snprintf(buf, sizeof(buf), "%s", "abcdefghij");


Kernel 指南：不要新增 strcpy()，有 FORTIFY/編譯器旗標也不是萬靈丹。 
infradead.org

strncpy(dst, src, n)（少用）

危險：如果 src 長度 ≥ n，dst 不保證有 '\0'；常被誤認為安全版 strcpy。

較安全用法

strncpy(dst, src, size - 1);
dst[size - 1] = '\0';   // 手動補 NUL


更佳替代：strlcpy() 或（kernel）strscpy()。

strcat(dst, src)（避免）

危險：不檢查剩餘空間，極容易溢位。

替代：strlcat() 或直接 snprintf() 重建整段字串。

strncat(dst, src, n)（易誤用）

陷阱：n 是「最多拷貝的字元數」，不是剩餘空間。

正確用法

strncat(dst, src, sizeof(dst) - strlen(dst) - 1);


更佳替代：strlcat()（會看整體目標大小）。

sprintf(dst, "...")（避免）

危險：無長度限制 → 溢位。

替代：snprintf(dst, size, ...)。

snprintf(dst, size, "...")（首選）

安全性：限制寫入長度並保證 NUL；回傳值是「應該要寫的長度（不含 NUL）」。

必要檢查

int need = snprintf(buf, sizeof(buf), "Hello %s", name);
if (need >= (int)sizeof(buf)) {
    // 內容被截斷，依情況擴充或處理
}

strlen(s) vs strnlen(s, max)

危險：strlen() 會一路找 '\0'，若缺 NUL → 讀爆。

替代：strnlen(s, max) 安全上限。

size_t n = strnlen(maybe_non_terminated, cap);

strlcpy(dst, src, size)（glibc 2.38+ 提供；舊版可用 libbsd）

優點：永遠 NUL 結尾；回傳 來源長度（可判斷是否截斷）。

用法

size_t need = strlcpy(dst, src, sizeof(dst));
if (need >= sizeof(dst)) { /* 截斷了 */ }


可用性：glibc 自 2.38 起內建（Fedora 新版都具備；更舊系統可連結 -lbsd）。 
GitHub
+1
Stack Overflow

strlcat(dst, src, size)

功能：基於整體 size 拼接，保證 NUL；回傳「欲得到的總長」。

用法

size_t need = strlcat(dst, tail, sizeof(dst));
if (need >= sizeof(dst)) { /* 被截斷 */ }


可用性：同上（glibc 2.38+）。 
GitHub

（Kernel）strscpy(dst, src, size)（最佳實務）

行為：永遠 NUL 結尾；若放不下，回傳 -E2BIG，否則回傳實際複製字元數。

用法

int n = strscpy(dst, src, size);
if (n < 0) { /* 截斷，採取動作 */ }


為何推薦：比 strncpy/strlcpy 更直覺的錯誤訊號與語意，Kernel 社群推薦。 
LWN.net
Debian Manpages

（Kernel）scnprintf(dst, size, "...")

差異：回傳「實際寫入長度」，比 snprintf 的「應寫入長度」更直覺。

用法

int wrote = scnprintf(buf, sizeof(buf), "val=%d", v);
// wrote <= size-1，且 buf 已 NUL 結尾

3) 常見「地雷場景」速查

沒有 '\0' 的「字串」：例如 strncpy 剛好寫滿或二進位資料誤當字串。→ 一律用 strnlen() 探測上限，或自行保證結尾。

UTF-8 誤用 strlen 當字數：會用位元組數當字數，導致 UI 切割錯亂或把一個多位元組字元切半。→ 需要「碼點」層級處理（mbrtowc/wcrtomb、iconv、或高階函式庫）。

strtok() 在多執行緒：共享靜態狀態、會改原字串。→ strtok_r()/strsep()。

memcpy 來源/目的重疊：未定義行為。→ memmove()。

printf("%s", user_input)：若 user_input 不是以 NUL 結尾，會讀爆。→ 先驗證長度或用 %.*s 搭配長度上限。

printf("%.*s\n", (int)strnlen(s, cap), s);

4) Fedora/Linux 實務清單
編譯建議

開啟警告與強化：

gcc -Wall -Wextra -Wconversion -Werror=format-security -fstack-protector-strong -D_FORTIFY_SOURCE=2 your.c


需要 GNU 擴充（如 getline）時加上：

#define _GNU_SOURCE
#include <stdio.h>


strlcpy/strlcat 可用性：Fedora 採用新 glibc，2.38+ 已內建；更舊環境可用 libbsd：

gcc your.c -lbsd   # 若目標環境沒有 glibc 2.38+


（Fedora Rawhide 曾在 glibc 2.37.9000 預告導入，2.38 正式提供。） 
GitHub
+1

常用 POSIX/GNU 工具

讀一行（會自動擴充）：getline(&buf, &cap, stdin)。

複製新字串：strdup()；用畢 free()。

分隔字串：strtok_r() 或 strsep()。

路徑組裝：盡量 snprintf() 一次成形，或用更高階庫（glib 的 GString、g_build_filename）。

5) 安全範本與範例
A. 安全拷貝（User space）
// 目標固定大小，不確定 src 長度
char dst[64];
size_t need = strlcpy(dst, src, sizeof(dst));
if (need >= sizeof(dst)) {
    // 被截斷，視需求處理（記錄、擴充、回報錯誤）
}

B. 安全拼接（User space）
char path[256] = "/home/user/";
size_t need = strlcat(path, filename, sizeof(path));
if (need >= sizeof(path)) {
    // 無法完整拼接
}

C. 安全格式化
char buf[128];
int need = snprintf(buf, sizeof(buf), "user=%s id=%u", user, id);
if (need >= (int)sizeof(buf)) {
    // 截斷
}

D. 動態讀入未知長度（建議）
#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>

char *line = NULL;
size_t cap = 0;
ssize_t n = getline(&line, &cap, stdin);
if (n >= 0) {
    // 使用 line（含 '\n'），用畢 free(line)
}
free(line);

E. Tokenize（可重入）
char *save, *tok;
for (tok = strtok_r(str, " \t", &save); tok; tok = strtok_r(NULL, " \t", &save)) {
    // 使用 tok
}

F. Kernel：安全拷貝與格式化
// 拷貝
int n = strscpy(dst, src, sizeof(dst));
if (n < 0) { /* 截斷 */ }

// 格式化
int wrote = scnprintf(buf, sizeof(buf), "val=%d", v);
// wrote: 實際寫入長度


（strscpy 永遠 NUL 結尾，放不下回 -E2BIG；scnprintf 回傳實寫入長度。） 
LWN.net
Debian Manpages

6) 「安全 vs 危險」一覽表（含要點）
函式	風險	正確姿勢 / 替代
strcpy	無界 → 溢位	別用；改 strlcpy / snprintf /（kernel）strscpy。 
infradead.org

strncpy	可能無 NUL；易誤用	若不得不用：n = size-1 並手動補 NUL；更好：strlcpy/strscpy。
strcat	無界拼接	別用；改 strlcat 或 snprintf。
strncat	n 易誤解	n = size - strlen(dst) - 1；更好：strlcat。
sprintf	無界格式化	別用；改 snprintf。
snprintf	可能截斷	檢查 ret >= size。
strlen	無上限讀取	用 strnlen(s, max)。
strlcpy/strlcat	舊系統可能沒有	glibc 2.38+ 內建；否則連 -lbsd。回傳需求長度用來檢測截斷。 
GitHub
Stack Overflow

strscpy（kernel）	—	永遠 NUL；不夠回 -E2BIG，檢查回傳值。 
LWN.net

scnprintf（kernel）	—	回傳實際寫入長度；比 snprintf 更直覺。
7) 進一步注意：多語系/UTF-8

切割或截斷時，避免把多位元組字元「切半」。

若要以「字元數」或「畫寬」計算，使用寬字元/多位元組 API（mbrtowc、wcslen、wcwidth 等），或高階函式庫（如 ICU、GLib）。

以「位元組」為單位的 API（幾乎整個 <string.h>）只知道 bytes，不知道「字」。

8) 最後清單（寫字串時逐條檢查）

我是否知道目標緩衝區的 完整大小，且 API 有用到它？

我是否 保證結尾 NUL？（或 API 會幫我保證？）

我有沒有 檢查回傳值（尤其 snprintf/strl*/strscpy）？

我是否誤把 位元組數當成「字數」？

若在 kernel：我是否使用 strscpy/scnprintf 並避免 strcpy？ 
infradead.org

參考（選讀）

glibc 自 2.38 起提供 strlcpy/strlcat（Fedora 新版可用；早期可以 libbsd 供應）。 
GitHub
+1

Linux kernel 推薦 strscpy，strcpy 被列為不應新增的介面。 
LWN.net
infradead.org

strscpy 的 man 與說明。 
Debian Manpages
man7.org