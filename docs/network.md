# Устройство сети RPi
Работа сети на **2017-11-29-raspbian-stretch-lite** поддерживается двумя предустановленными службами:
* **networking** - служба включает все сетевые интерфейсы в момент запуска [5].
* **dhcpcd** - служба обеспечивает настройку адресации и маршрутризации на интерфейсах, полученных динамически или указаных в файле настроек статически.

> Для работы в режиме роутера (точки доступа) RPi необходим dhcp-сервер. Он служит для автоматической выдачи настроек текущей сети подключившимся клиентам. В роли такого сервера может выступать `isc-dhcp-server` или `dnsmasq`.

## dhcpcd
Начиная с Raspbian Jesse настройки сети больше не задаются в файле `/etc/network/interfaces`. Теперь за выдачу адресации и настройку маршрутизации отвечает `dhcpcd` [4].

По умолчанию на всех интерфейсах включен dhcp-клиент. Настройки интерфейсов меняются в файле `/etc/dhcpcd.conf`. Для того, чтобы поднять точку доступа необходимо прописать статический ip-адрес. Для этого в конец файла необходимо добавить следующие строки:

```
interface wlan0
static ip_address=192.168.11.1/24
```

> Если интерфейс является беспроводным (wlan), то служба `dhcpcd` триггерит `wpa_supplicant` [13], который в свою очередь работает непосредственно с wifi-адаптером и переводит его в заданное состояние.

## wpa_supplicant
**wpa_supplicant** - служба конфигурирует wifi-адаптер. Служба wpa_supplicant работает не как самостоятельная (хотя как таковая существует), а запускается как дочерний процесс от `dhcpcd` [13].

```
Aug 25 08:32:28 raspberrypi dhcpcd[342]: wlan0: starting wpa_supplicant
Aug 25 08:32:28 raspberrypi dhcpcd-run-hooks[388]: wlan0: starting wpa_supplicant
```

```
pi@clever-smirnov:~ $ systemctl status dhcpcd
● dhcpcd.service - dhcpcd on all interfaces
   Loaded: loaded (/lib/systemd/system/dhcpcd.service; enabled; vendor preset: enabled)
  Drop-In: /etc/systemd/system/dhcpcd.service.d
           └─wait.conf
   Active: active (running) since Fri 2018-05-18 10:57:45 UTC; 1 weeks 4 days ago
  Process: 341 ExecStart=/usr/lib/dhcpcd5/dhcpcd -q -w (code=exited, status=0/SUCCESS)
 Main PID: 498 (dhcpcd)
   CGroup: /system.slice/dhcpcd.service
           ├─444 wpa_supplicant -B -c/etc/wpa_supplicant/wpa_supplicant.conf -iwl
           └─498 /sbin/dhcpcd -q -w
```

```(bash)
pi@clever-smirnov:~ $ ps aux | grep wpa_supplicant
root       444  0.0  0.3  10136  3552 ?        Ss   06:46   0:00 wpa_supplicant -B -c/etc/wpa_supplicant/wpa_supplicant.conf -iwlan0 -Dnl80211,wext
```

```
pi@clever-smirnov:~ $ sudo find /etc -name "*wpa*" -exec file {} \;

# /etc/wpa_supplicant/ifupdown.sh работает, если wpa_supplicant запущена как самостоятельная служба
/etc/network/if-up.d/wpasupplicant: symbolic link to ../../wpa_supplicant/ifupdown.sh
/etc/network/if-pre-up.d/wpasupplicant: symbolic link to ../../wpa_supplicant/ifupdown.sh
/etc/network/if-down.d/wpasupplicant: symbolic link to ../../wpa_supplicant/ifupdown.sh
/etc/network/if-post-down.d/wpasupplicant: symbolic link to ../../wpa_supplicant/ifupdown.sh

# Action script to enable/disable wpa-roam interfaces in reaction to ifplugd events.
/etc/ifplugd/action.d/action_wpa: symbolic link to ../../wpa_supplicant/action_wpa.sh
/etc/wpa_supplicant/action_wpa.sh: POSIX shell script, ASCII text executable

# Настройки и скрипты для запуска wpa_supplicant располагаются в папке /etc/wpa_supplicant
/etc/wpa_supplicant: directory

# Дефолтный конфигурационный файл
/etc/wpa_supplicant/wpa_supplicant.conf: ASCII text

/etc/dbus-1/system.d/wpa_supplicant.conf: exported SGML document, ASCII text
```

Конфиг запускаемый от `wpa_supplicant` который триггерится `dhcpcd`, должен иметь имя `wpa_supplicant.conf`. Внутри конфига указываются общие настройки и параметры для адаптера. Подробнее о синтаксисе `wpa_supplicant.conf` [TODO WIKI]

Ниже, располагаются секции `network` - основные настройки wifi-сети такие как SSID, пароль, режим работы адаптера. Таких блоков может быть несколько, но используется первый рабочий. Например, если вы указали в первом блоке подключение к некоторой недоступной сети, то адаптер будет настроен следующей удачной секцией, если такая есть.

Для создания секции `network` можно воспользоваться утилитой `wpa_passphrase`:

 ```bash
  wpa_passphrase SSID PASSWD
  network={
   ssid="SSID"
   #psk="PASSWD"
   psk=7a1769b3cb333afd729ad80f3da6bc3b08999f4e923de19eb943b8753c761c72
  }
  ```
  > После генерации секции можно удалить закоментированное поле `psk` и оставить только поле с хешем пароля, либо наоборот.

Пример `wpa_supplicant.conf`
```
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1
country=GB

network={
        ssid="CLEVER-SMIRNOV"
        psk="cleverwifi"
        mode=2
        proto=RSN
        key_mgmt=WPA-PSK
        pairwise=CCMP
        group=CCMP
        auth_alg=OPEN
}
```

### Несколько wifi-адаптеров
В системе может быть несколько wifi-адаптеров. Если для них корректно подключены драйвера, то их можно увидеть вызвав `ifconfig` (например wlan0 и wlan1).

Если у вас несколько адаптеров в этом случае для всех будет использоваться одна и таже самая рабочая секция `network`. Это связано с тем что для каждого отдельного интерфейса `dhcpcd` отдельно создает по дочернему процессу `wpa_supplicant` в котором выполняется один тот же код (т.к. конфиг один и тот же).

Для работы нескольких адаптеров с отдельными настройками для каждого, в стандартном вызываемом скрипте `dhcpcd` реализован механизм запуска разных конфигурационных скриптов. Для его использования необходимо переименовать стандартный файл конфига по следующему образцу: `wpa_supplicant-<имя интерфейса>.conf`, например `wpa_supplicant-wlan0.conf`.

Для применения настроек необходимо перезапустить родительский процесс - службу `dhcpcd`. Сделать это можно следующей командой:
```bash
sudo systemctl restart dhcpcd
```
## dhcp-server

### dnsmasq

```bash
sudo apt install dnsmasq
```

```bash
echo "
interface=wlan0
address=/clever/coex/192.168.11.1
dhcp-range=192.168.11.100,192.168.11.200,12h
no-hosts
filterwin2k
bogus-priv
domain-needed
quiet-dhcp6
}" >> /etc/dnsmasq.conf
```

### dnsmasq-base
Можно использовать dnsmasq-base - консольная утилита, не являющаяся службой в отличие от dnsmasq.

```bash
# Вызов dnsmasq-base
sudo dnsmasq --interface=wlan0 --address=/clever/coex/192.168.11.1 --no-daemon --dhcp-range=192.168.11.100,192.168.11.200,12h --no-hosts --filterwin2k --bogus-priv --domain-needed --quiet-dhcp6 --log-queries

# Подробнее о dnsmasq-base
dnsmasq --help

# или
man dnsmasq
```

### isc-dhcp-server

```bash
sudo apt install isc-dhcp-server
```

## References

1. [habr.com: Linux WiFi из командной строки с wpa_supplicant](https://habr.com/post/315960/)
2. [wiki.archlinux.org: WPA supplicant (Русский)](https://wiki.archlinux.org/index.php/WPA_supplicant_(Русский))
3. [blog.hoxnox.com: WiFi access point with wpa_supplicant](http://blog.hoxnox.com/gentoo/wifi-hotspot.html)
4. [dmitrysnotes.ru: Raspberry Pi 3. Присвоение статического IP-адреса](http://dmitrysnotes.ru/raspberry-pi-3-prisvoenie-staticheskogo-ip-adresa)
5. [thegeekdiary.com: Linux OS Service ‘network’](https://www.thegeekdiary.com/linux-os-service-network/)
6. [frillip.com: USING YOUR NEW RASPBERRY PI 3 AS A WIFI ACCESS POINT WITH HOSTAPD](https://frillip.com/using-your-raspberry-pi-3-as-a-wifi-access-point-with-hostapd/) (также здесь есть инструкция по настройке форвардинга для использования RPi в качестве шлюза для выхода в интернет)
7. [habr.com: Настраиваем ddns-сервер на GNU/Linux Debian 6](https://habr.com/sandbox/30433/) (Хорошая статья по настройке ddns-сервера на основе `bind` и `isc-dhcp-server`)
8. [pro-gram.ru: Установка и настройка DHCP сервера на Ubuntu 16.04.](https://pro-gram.ru/dhcp-server-ubuntu.html) (Настройка isc-dhcp-server)
9. [expert-orda.ru: Настройка DHCP-сервера на Ubuntu](http://expert-orda.ru/posts/liuxnewbie/125--dhcp-ubuntu) (Настройка isc-dhcp-server)
10. [academicfox.com: Raspberry Pi беспроводная точка доступа (WiFi access point)](http://academicfox.com/raspberry-pi-besprovodnaya-tochka-dostupa-wifi-access-point/) (Настройка маршрутов, hostapd, isc-dhcp-server)
11. [weworkweplay.com: Automatically connect a Raspberry Pi to a Wifi network](http://weworkweplay.com/play/automatically-connect-a-raspberry-pi-to-a-wifi-network/) (Есть настройки для создания открытой точки доступа)
12. [wiki.archlinux.org: WPA supplicant](https://wiki.archlinux.org/index.php/WPA%20supplicant)
13. [wiki.archlinux.org: dhcpcd](https://wiki.archlinux.org/index.php/Dhcpcd#10-wpa_supplicant) (dhcpcd hook wpa_supplicant)
