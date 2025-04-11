# soal_1
### a) Downloading the Clues
pertama-tama kita membuat file action.c yang dimana kalau dijalankan tanpa argumen tambahan akan mendownload dan langsung meng unzip file tersebut. Dan tambahkan juga fungsi agar menghapus file Clues.zip setelah diekstrak dan semisal saat dijalankan folder Clues sudah ada, maka tidak akan mendownload Clues.zip lagi
```C
void download_and_unzip() {
    if (access("Clues", F_OK) == 0) {
        printf("[!] Folder Clues sudah ada. Lewati download.\n");
        return;
    }

    pid_t pid = fork();
    if (pid == 0) {
        char *argv[] = {"wget", "--no-check-certificate", "-O", "Clues.zip",
                        "https://drive.google.com/uc?id=example&export=download", NULL};
        execvp("wget", argv);
        perror("wget gagal");
        exit(1);
    } else {
        wait(NULL);
    }

    pid_t pid2 = fork();
    if (pid2 == 0) {
        char *argv[] = {"unzip", "Clues.zip", NULL};
        execvp("unzip", argv);
        perror("unzip gagal");
        exit(1);
    } else {
        wait(NULL);
    }

    remove("Clues.zip");
    printf("[+] Clues.zip berhasil diunduh dan diekstrak.\n");
}
```

### b) Filtering the Files
