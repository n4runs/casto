## Soal 3 - The Lost Dungeon
> Soal ini terdapat revisi pada bagian battle mode.

Author : Naruna Vicranthyo Putra Gangga (5027241105)

### Deskripsi
Author diberikan tugas untuk membuat yang terdiri dari dua komponen utama:
- dungeon.c sebagai server yang menjalankan seluruh logika permainan.
- player.c sebagai client yang berinteraksi dengan pemain melalui koneksi socket TCP/IP.
Pemain dapat terhubung ke server untuk menelusuri dungeon, membeli senjata, mengecek status, serta bertarung melawan musuh.

---

## Entering the Dungeon

File `dungeon.c` berperan sebagai **server** yang menerima koneksi dari **client** (`player.c`) melalui komunikasi socket. Semua perintah dikirim dari `player.c` ke `dungeon.c`, lalu diproses oleh server.

> Server mendukung **multi-client** (lebih dari satu pemain dapat bermain secara bersamaan).
```C
//dungeon.c
void *handleClient(void *arg) {
    int clientSock = *(int *)arg;
    free(arg);

    struct Player player;
    player.socket = clientSock;
    player.gold = 100;
    player.baseDamage = 10;
    player.inventoryCount = 0;
    player.enemyDefeated = 0;
    strcpy(player.equipped.name, "Fists");
    player.equipped.damage = 0;
    strcpy(player.equipped.passive, "-");

    char buffer[1024];

    while (1) {
        int valread = read(clientSock, buffer, 1024);
        if (valread <= 0) break;
        buffer[valread] = 0;
        buffer[strcspn(buffer, "\r\n")] = 0;

        if (strcmp(buffer, "1") == 0) {
            char response[512];
            showStats(&player, response);
            send(clientSock, response, strlen(response), 0);
        } else if (strcmp(buffer, "2") == 0) {
            shop(&player, buffer, clientSock);
        } else if (strcmp(buffer, "3") == 0) {
            char response[1024];
            showInventory(&player, response);
            send(clientSock, response, strlen(response), 0);
            int val = read(clientSock, buffer, 1024);
            buffer[val] = 0;
            buffer[strcspn(buffer, "\r\n")] = 0;
            int choice = atoi(buffer);
            if (choice > 0 && choice <= player.inventoryCount) {
                player.equipped = player.inventory[choice - 1];
                send(clientSock, "Weapon equipped!\n", 17, 0);
            } else {
                send(clientSock, "Cancelled.\n", 11, 0);
            }
        } else if (strcmp(buffer, "4") == 0) {
            battle(&player, buffer, clientSock);
        } else if (strcmp(buffer, "5") == 0) {
            send(clientSock, "Goodbye!\n", 9, 0);
            break;
        } else {
            send(clientSock, "Invalid option.\n", 16, 0);
        }
    }

    close(clientSock);
    return NULL;
}

int main() {
    int server_fd, new_socket;
    struct sockaddr_in address;
    int opt = 1;
    socklen_t addrlen = sizeof(address);

    server_fd = socket(AF_INET, SOCK_STREAM, 0);
    setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    address.sin_family = AF_INET;
    address.sin_addr.s_addr = INADDR_ANY;
    address.sin_port = htons(PORT);

    bind(server_fd, (struct sockaddr *)&address, sizeof(address));
    listen(server_fd, MAX_CLIENTS);

    printf("Dungeon server running on port %d...\n", PORT);

    while (1) {
        new_socket = accept(server_fd, (struct sockaddr *)&address, &addrlen);
        int *pclient = malloc(sizeof(int));
        *pclient = new_socket;
        pthread_t tid;
        pthread_create(&tid, NULL, handleClient, pclient);
    }

    return 0;
}
```
---

## Sightseeing

Saat `player.c` dijalankan, pemain akan terhubung ke server dan ditampilkan menu utama (main menu). Menu ini mencakup pilihan:

1. Show Player Stats  
2. Shop (Buy Weapons)  
3. View Inventory & Equip Weapons  
4. Battle Mode  
5. Exit Game

Menu ditampilkan dalam tampilan terminal interaktif menggunakan warna (ANSI escape codes).
```C
//player.c
void strip_ansi(char *str) {
    regex_t regex;
    regcomp(&regex, "\033\\[[0-9;]*m", REG_EXTENDED);
    regmatch_t match;
    char result[BUFFER_SIZE] = "";
    char *cursor = str;

    while (regexec(&regex, cursor, 1, &match, 0) == 0) {
        strncat(result, cursor, match.rm_so);
        cursor += match.rm_eo;
    }
    strcat(result, cursor);
    strcpy(str, result);
    regfree(&regex);
}

void dashboard() {
    printf("\033[2J\033[1;1H");
    printf("\n");
    printf("\033[1;36m");
    printf("████████╗██╗  ██╗███████╗    ██╗      ██████╗ ███████╗████████╗    ██████╗ ██╗   ██╗███╗   ██╗ ██████╗ ███████╗ ██████╗ ███╗   ██╗\n");
    printf("╚══██╔══╝██║  ██║██╔════╝    ██║     ██╔═══██╗██╔════╝╚══██╔══╝    ██╔══██╗██║   ██║████╗  ██║██╔════╝ ██╔════╝██╔═══██╗████╗  ██║\n");
    printf("   ██║   ███████║█████╗      ██║     ██║   ██║███████╗   ██║       ██║  ██║██║   ██║██╔██╗ ██║██║  ███╗█████╗  ██║   ██║██╔██╗ ██║\n");
    printf("   ██║   ██╔══██║██╔══╝      ██║     ██║   ██║╚════██║   ██║       ██║  ██║██║   ██║██║╚██╗██║██║   ██║██╔══╝  ██║   ██║██║╚██╗██║\n");
    printf("   ██║   ██║  ██║███████╗    ███████╗╚██████╔╝███████║   ██║       ██████╔╝╚██████╔╝██║ ╚████║╚██████╔╝███████╗╚██████╔╝██║ ╚████║\n");
    printf("   ╚═╝   ╚═╝  ╚═╝╚══════╝    ╚══════╝ ╚═════╝ ╚══════╝   ╚═╝       ╚═════╝  ╚═════╝ ╚═╝  ╚═══╝ ╚═════╝ ╚══════╝ ╚═════╝ ╚═╝  ╚═══╝\n\n");
    printf("\033[0m");
}

void print_menu() {
    dashboard();
    printf("\x1b[32m=== MAIN MENU ===\x1b[0m\n");
    printf("1. Show Player Stats\n");
    printf("2. Shop (Buy Weapons)\n");
    printf("3. View Inventory & Equip Weapons\n");
    printf("4. Battle Mode\n");
    printf("5. Exit Game\n");
    printf("\x1b[33mChoose an option: \x1b[0m");
}
```

![image](https://github.com/user-attachments/assets/b3c9c25d-3326-4596-ab6b-85732cc6a474)

---

## Status Check

Jika opsi **Show Player Stats** dipilih, maka akan ditampilkan informasi berikut:

- Jumlah gold yang dimiliki
- Senjata yang sedang digunakan
- Base Damage
- Jumlah musuh yang telah dikalahkan
- Jika senjata memiliki **Passive**, akan ditampilkan juga efek tersebut
```C
//dungeon.c
struct Player {
    int socket;
    int gold;
    int baseDamage;
    struct Weapon equipped;
    struct Weapon inventory[MAX_INVENTORY];
    int inventoryCount;
    int enemyDefeated;
};

void showStats(struct Player *p, char *buffer) {
    int totalDamage = p->baseDamage + p->equipped.damage;
    sprintf(buffer,
        "\n\x1b[32m== Player Stats ==\x1b[0m\n"
        "\x1b[33mGold: \x1b[0m%d\n"
        "\x1b[31mBase Damage: \x1b[0m%d\n"
        "\x1b[31mWeapon Damage: \x1b[0m%d\n"
        "\x1b[36mTotal Damage: \x1b[0m%d\n"
        "\x1b[34mEquipped Weapon: \x1b[0m%s\n"
        "\x1b[35mPassive Skill: \x1b[0m%s\n"
        "\x1b[35mEnemies Defeated: \x1b[0m%d\n",
        p->gold,
        p->baseDamage,
        p->equipped.damage,
        totalDamage,
        p->equipped.name,
        p->equipped.passive,
        p->enemyDefeated
    );
}
```

![image](https://github.com/user-attachments/assets/f32dd46f-f6e8-4ecc-b2a7-b5886ed0c159)

---

## Weapon Shop

Saat opsi **Shop** dipilih, pemain akan melihat daftar senjata yang dapat dibeli. Setiap senjata memiliki:

- Nama
- Harga (dalam gold)
- Damage
- Passive (jika ada)

> Terdapat **minimal 5 senjata**, dan **minimal 2** di antaranya memiliki efek **Passive** unik (contoh: critical chance, insta kill).

Semua logika dan data senjata terdapat di dalam `shop.h`, yang digunakan oleh `dungeon.c` (tanpa membutuhkan `shop.c` lagi).
```C
//dungeon.c
void shop(struct Player *p, char *buffer, int clientSock) {
    char shopDisplay[BUFFER_SIZE];
    showShop(shopDisplay);
    send(clientSock, shopDisplay, strlen(shopDisplay), 0);
    int valread = read(clientSock, buffer, BUFFER_SIZE);
    buffer[valread] = '\0';
    buffer[strcspn(buffer, "\r\n")] = 0;

    int choice = atoi(buffer);
    if (choice < 1 || choice > MAX_WEAPONS) {
        send(clientSock, "Invalid choice.\n", 16, 0);
        return;
    }

    int result = buyWeapon(choice - 1, &p->gold, p->inventory, &p->inventoryCount);
    if (result == 1) {
        char msg[128];
        snprintf(msg, sizeof(msg), "Purchased %s!\n", shopWeapons[choice - 1].name);
        send(clientSock, msg, strlen(msg), 0);
    } else if (result == -1) {
        send(clientSock, "Not enough gold.\n", 17, 0);
    } else if (result == -2) {
        send(clientSock, "Inventory is full.\n", 20, 0);
    } else {
        send(clientSock, "Invalid weapon.\n", 17, 0);
    }
}
```

![image](https://github.com/user-attachments/assets/f9dfbbc7-b778-4a79-be9c-d154fbe0352c)

---

## Handy Inventory

Setelah membeli senjata, pemain dapat memilih **View Inventory & Equip Weapons** untuk:

- Melihat semua senjata yang dimiliki
- Mengganti senjata yang sedang digunakan

Senjata yang sedang digunakan akan mempengaruhi output pada **Show Player Stats**, termasuk damage dan passive.
```C
//dungeon.c
void showInventory(struct Player *p, char *buffer) {
    strcpy(buffer, "\n== Inventory ==\n");
    for (int i = 0; i < p->inventoryCount; i++) {
        char line[256];
        sprintf(line, "%d. %s (%d dmg, %s)%s\n",
                i + 1,
                p->inventory[i].name,
                p->inventory[i].damage,
                p->inventory[i].passive,
                strcmp(p->inventory[i].name, p->equipped.name) == 0 ? " [Equipped]" : "");
        strcat(buffer, line);
    }
    strcat(buffer, "Choose item number to equip or 0 to cancel: ");
}
```

![image](https://github.com/user-attachments/assets/e6c60c40-7b3f-4d9b-b0f6-84369045cbbd)

---

## Enemy Encounter

Jika pemain memilih **Battle Mode**, maka:

- Musuh akan muncul dengan jumlah darah (HP) acak, misalnya 50–200 HP
- Ditampilkan health-bar visual dan angka HP
- Pemain dapat memasukkan perintah:
  - `attack` → untuk menyerang
  - `exit` → untuk keluar dari Battle Mode

Setelah musuh dikalahkan, pemain akan mendapatkan **reward gold acak**, dan musuh baru akan muncul.
```C
//dungeon.c
void battle(struct Player *p, char *buffer, int clientSock) {
    while (1) {
        int enemyHP = (rand() % 151) + 50;
        int maxHP = enemyHP;

        char intro[256];
        sprintf(intro,
            "\n\x1b[31m=== BATTLE STARTED ===\x1b[0m\n"
            "Enemy appeared with:\n"
            "\x1b[42m[                     ]\x1b[0m %d/ %d HP\n"
            "Type '\x1b[32mattack\x1b[0m' to attack or '\x1b[31mexit\x1b[0m' to leave battle.\n",
            enemyHP, enemyHP);
        send(clientSock, intro, strlen(intro), 0);

        while (enemyHP > 0) {
            int valread = read(clientSock, buffer, 1024);
            if (valread <= 0) return;
            buffer[valread] = 0;
            buffer[strcspn(buffer, "\r\n")] = 0;

            if (strcmp(buffer, "exit") == 0) {
                send(clientSock, "Exiting Battle Mode.\n", 22, 0);
                return;
            } else if (strcmp(buffer, "attack") != 0) {
                send(clientSock, "Invalid command. Type 'attack' or 'exit'.\n", 42, 0);
                continue;
            }

            int instakill = 0, crit = 0, doubleGold = 0, charm = 0, ghost = 0;
            char msg[1024] = "";

            if (strstr(p->equipped.passive, "insta-kill") && rand() % 100 < 10) instakill = 1;
            if (strstr(p->equipped.passive, "crit") && rand() % 100 < 30) crit = 1;
            if (strstr(p->equipped.passive, "double gold") && rand() % 100 < 20) doubleGold = 1;
            if (strstr(p->equipped.passive, "memikat hati") && rand() % 100 < 25) charm = 1;
            if (strstr(p->equipped.passive, "ghosting") && rand() % 100 < 25) ghost = 1;

            int base = p->baseDamage + p->equipped.damage;
            int bonus = rand() % 6;
            int dmg = base + bonus;

            if (instakill) {
                enemyHP = 0;
                snprintf(msg, sizeof(msg),
                    "\x1b[35m=== INSTANT KILL! ===\x1b[0m\n"
                    "Your \x1b[36m%s\x1b[0m obliterated the enemy instantly!\n",
                    p->equipped.name);
            } else {
                if (crit) {
                    dmg *= 2;
                    snprintf(msg, sizeof(msg),
                        "\x1b[33m=== CRITICAL HIT! ===\x1b[0m\n"
                        "You dealt \x1b[31m%d damage\x1b[0m!\n", dmg);
                } else {
                    snprintf(msg, sizeof(msg),
                        "You dealt \x1b[31m%d damage\x1b[0m!\n", dmg);
                }
                enemyHP -= dmg;
                if (enemyHP < 0) enemyHP = 0;
            }

            if (charm)
                strcat(msg, "\x1b[35m[Passive Activated: You charmed the enemy, reducing their will to fight!]\x1b[0m\n");
            if (ghost)
                strcat(msg, "\x1b[35m[Passive Activated: You ghosted the enemy, it hesitated!]\x1b[0m\n");

            char healthStatus[128];
            getEnemyStatusBar(healthStatus, enemyHP, maxHP);
            strcat(msg, healthStatus);

            if (enemyHP <= 0) {
                int reward = (rand() % 51) + 50;
                if (doubleGold) reward *= 2;
                p->gold += reward;
                p->enemyDefeated++;

                char rewardMsg[128];
                sprintf(rewardMsg,
                    "\n\x1b[32m=== REWARD ===\x1b[0m\n"
                    "You earned \x1b[33m%d gold\x1b[0m!\n", reward);
                strcat(msg, rewardMsg);

                send(clientSock, msg, strlen(msg), 0);
                break;
            }

            send(clientSock, msg, strlen(msg), 0);
        }
    }
}
```

![image](https://github.com/user-attachments/assets/9a8df8e7-49d8-4292-9b5d-cf31c82beaa4)

---

## Other Battle Logic

### Damage Equation

- Damage dasar ditentukan dari base damage senjata
- Variasi damage ditambahkan dengan bilangan acak
- Terdapat kemungkinan **Critical Hit** yang menggandakan damage

### Passive

Jika senjata memiliki efek Passive, efek tersebut akan aktif sesuai logika masing-masing. Contoh:

- Passive: `+30% crit chance` → meningkatkan peluang critical hit
- Passive: `+10% insta kill` → memiliki peluang untuk langsung mengalahkan musuh

Saat passive aktif, akan ditampilkan notifikasi efek tersebut di terminal.
```C
//dungeon.c
            int instakill = 0, crit = 0, doubleGold = 0, charm = 0, ghost = 0;
            char msg[1024] = "";

            if (strstr(p->equipped.passive, "insta-kill") && rand() % 100 < 10) instakill = 1;
            if (strstr(p->equipped.passive, "crit") && rand() % 100 < 30) crit = 1;
            if (strstr(p->equipped.passive, "double gold") && rand() % 100 < 20) doubleGold = 1;
            if (strstr(p->equipped.passive, "memikat hati") && rand() % 100 < 25) charm = 1;
            if (strstr(p->equipped.passive, "ghosting") && rand() % 100 < 25) ghost = 1;

            int base = p->baseDamage + p->equipped.damage;
            int bonus = rand() % 6;
            int dmg = base + bonus;

            if (instakill) {
                enemyHP = 0;
                snprintf(msg, sizeof(msg),
                    "\x1b[35m=== INSTANT KILL! ===\x1b[0m\n"
                    "Your \x1b[36m%s\x1b[0m obliterated the enemy instantly!\n",
                    p->equipped.name);
            } else {
                if (crit) {
                    dmg *= 2;
                    snprintf(msg, sizeof(msg),
                        "\x1b[33m=== CRITICAL HIT! ===\x1b[0m\n"
                        "You dealt \x1b[31m%d damage\x1b[0m!\n", dmg);
                } else {
                    snprintf(msg, sizeof(msg),
                        "You dealt \x1b[31m%d damage\x1b[0m!\n", dmg);
                }
                enemyHP -= dmg;
                if (enemyHP < 0) enemyHP = 0;
            }

            if (charm)
                strcat(msg, "\x1b[35m[Passive Activated: You charmed the enemy, reducing their will to fight!]\x1b[0m\n");
            if (ghost)
                strcat(msg, "\x1b[35m[Passive Activated: You ghosted the enemy, it hesitated!]\x1b[0m\n");
```
---

## Error Handling

Semua input dari pemain akan divalidasi. Jika input tidak valid (misal: memilih menu yang tidak ada atau salah memasukkan indeks senjata), maka akan ditampilkan pesan kesalahan yang sesuai.

---

## Struktur akhir
![image](https://github.com/user-attachments/assets/d9fa6824-c0ed-4ab8-9cc9-38f28180cdc7)


Kendala: Pada bagian `battle mode`, dimana yang seharusnya muncul enemy baru setelah mengalahkan enemy, melainkan malah balik ke main menu.
