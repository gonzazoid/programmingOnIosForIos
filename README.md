# programmingOnIosForIos
Программирование под ios на ios

Здравствуйте.

В статье рассказано как подготовить jailbroken ipad (ios7/8) для компиляции проектов на C/Objective C/C++ и рассмотрена компиляция одной программы (форк MobileTerminal).

Написано по мотивам статьи <a href="http://habrahabr.ru/sandbox/35151/">Компилируем C/C++ на iPhone/iPad</a> 2011 года, автор которой к сожалению так и не получил инвайта. Информация обновлена в соответствии с реалиями, gcc заменен на clang.

Итак, поехали.
Все описанное происходило на ipad2, ios7.1.1, позже - на ios8.1.2.
Предполагается что устройство уже jailbreak-нуто, Cydia установлена.

Dos-way, рассматриваемый в упомянутой статье рассматривать не будем, перейдем сразу к unix-way.

Сразу обозначу распространенное на просторах интернета заблуждение. SSH, который все настойчиво рекомендуют ставить сразу после jailbreak-а - не нужен. Во всех статьях, которые мне попадались на эту тему, рекомендовалось поднять локально ssh-сервер и локально же коннектиться к нему с ssh-клиента. Если вам не нужен доступ к вашему устройству снаружи - не делайте этого, достаточно MobileTerminal. Он монструозен, задумчив и немного amnesiac, но ниже рассматривается курс лечения, ничего страшного.

Для начала установим необходимый софт из Cydia. Нам нужны пакеты:
<ul>
<li>Core utilites(/bin)</li>
<li>Core utilites</li>
<li>ios toolchain (ставит LLVM+Clang, ld, cc tools, etc)</li>
<li>MobileTerminal либо его форк MTerminal (без разницы, потом все равно установим свой самосбор)</li>
<li>Vi Improved (vim)</li>
<li>wget</li>
<li>make</li>
<li>gawk</li>
<li>Debian Packager</li>
</ul>
После установки MobileTerminal меняем дефолтный пароль. В системе два пользователя - mobile и root, mobile для всех программ в системе, root - он и на ios root. У рута дефолтный пароль - alpine, делаем passwd и указываем свой пароль. Шелл у нас, кстати - bash (в cydia можно скачать и zsh, возможно есть и другие шеллы).

Теперь можно скачать SDK. Прямых линков на сайте apple я не нашел - они sdk раздают вместе с xCode и через свой store, поэтому пришлось погуглить. Неплохая подборка sdk разных версий есть здесь <a href="http://iphone.howett.net/sdks/">http://iphone.howett.net/sdks/</a>. Я для sdk завел директорию в домашней директории пользователя - SDKs, ставить sdk будем в нее. Ахтунг - я везде использую SDK семерки, при сборе с хидерами и либами восьмерки все работает крайне нестабильно. Делаем:

```bash
cd /User/SDKs
wget http://iphone.howett.net/sdks/dl/iPhoneOS7.1.sdk.tbz2
tar -xjf iPhoneOS7.1.sdk.tbz2
mv iPhoneOS7.1.sdk 7.1
```
в итоге путь до SDK получился такой: /User/SDKs/7.1/
Конкретно в этой версии(что будет в более поздних - не знаю) apple забыла положить crt_externs.h, но на сайте у них этот файлик есть. Исправляем несправедливость:

```bash
cd /User/SDKs/7.1/usr/include/
wget http://www.opensource.apple.com/source/Libc/Libc-391/include/crt_externs.h?txt -O crt_externs.h 
```

Итак, компилятор стоит, тулчаин присутствует, пора хэлловордить. Набиваем в vim-e:

```C++
#include <stdio.h>
int main(){
    printf("hello, word\n");
    return 0;
}
```
сохраняем в hello.c и собираем:<pre>clang hello.c -o hello -isysroot /User/SDKs/7.1/ -Xlinker -isysroot /User/SDKs/7.1 </pre>
Запускаем: ./hello Ок, все работает. Я у себя создал папку bin в домашней директории и прописал ее в PATH (vim /etc/profile), чтобы не заморачиваться с путями для самособранных велосипедов. Под семеркой этотработало, но вот восьмерка при вызове такой программы без полного пути почему то уходит в ребут, с этим еще разбираюсь.
Что же, теперь можно заняться нашим терминалом. MobileTerminal имеет плохую привычку - при переключении на другое приложение он напрочь забывает все что было и при возврате к его окну хочет начинать все заново - история сбрасывается. Во первых это напрягает, во вторых течет он не по детски, мне на ipad2 64gb приходилось перезапускать приложение каждые полчаса. 
MobileTerminal opensours-ный, но написан (на Objective-C) под xCode, которого у нас на девайсе нет. Но! нашлись <a href="https://bitbucket.org/lordscotland">добрые люди</a> и <a href="https://bitbucket.org/lordscotland/mterminal/src">переписали этот проект</a> под фрейворк TheOS, заодно убрав лишние плюшки, а именно - запуск нескольких сеансов шелла в одном окне (правильно это делать через мультиплексор, такой как tmux, например), убрали окно для добавления esc-последовательностей, реализовав вместо этого функционал кнопки ctrl, что позволяет ввести любой esc-символ напрямую с клавиатуры. Код также открыт. Но и здесь есть но! TheOS - совершенно лишняя сущность. Плюс у бинарника - клавиатура iphone, что для планшета вообще некомильфо. Плюс при landscape ориентации устройства при запуске программы она вылетает с негодованием. Ну и еще он выводит статус бар поверх приглашения шелла и это некрасиво.
Мы пойдем дальше - отучим его от TheOS, прикрутим нормальный make и назовем vTerminal-ом.
Отучить проект от TheOS оказалось нетрудно - пришлось просто пройтись по файлам проекта и прописать includ-ы соотв. хидеров. Заострять не буду, желающие могут скачать исходники обоих проектов и сравнить. Вот прямой линк на скачивание <a href="https://bitbucket.org/lordscotland/mterminal/get/b32cfdf343ae.zip">MTerminal</a>.
Теперь, что бы скомпилировать это добро, надо все *.m файлы скомпилировать в объектники, а объектники собрать в один бинарник. Делаем простой Makefile:

```CMake
SYSROOT = /User/SDKs/7.1
SOURCES    = $(wildcard *.m)
PRODUCT_NAME = vTerminal
CC = clang

CFLAGS = -Wall -v -isysroot $(SYSROOT)

LD = ${CC}
LDFLAGS = \
          -isysroot $(SYSROOT)\
          -ObjC \
          -framework CoreFoundation \
          -framework Foundation \
          -framework UIKit \
          -framework CoreText \
          -framework CoreGraphics

EXECUTABLE_NAME = $(PRODUCT_NAME)

OBJECTS = $(patsubst %.m,%.o,$(filter %.m,$(SOURCES)))

$(PRODUCT_NAME): $(OBJECTS)
	$(LD) $(LDFLAGS) -o $(PRODUCT_NAME) $(OBJECTS)
%.o: %.m
	$(CC) $(CFLAGS) -c $< -o $@
```
На что обратить внимание? Ну, во первых SYSROOT, значение которого используется во флаге isysroot для clang - указывает на директорию с SDK. Это гораздо лаконичнее, чем указывать пути хидеров для компилятора и либы для линкера. Да, путь прибит гвоздями к коду, но заморачиваться на configure в этой ситуации я считаю совершенно излишне. Если вы разместили sdk в другом месте - просто поменяйте путь. Я же планирую оформить sdk в виде .deb пакета и в будущем не заморачиваться на пути а просто прописывать его в depends-ах. 
Также стоит обратить внимание на флаги -framework - у нас Objective-C и apple, так что приходится указывать фреймворки, используемые проектом. Для нежелающих вникать - framework в исполнении apple - это типа либы, только навороченнее. Подробнее можно выяснить на <a href="https://developer.apple.com/library/ios/documentation/MacOSX/Conceptual/BPFrameworks/Frameworks.html">сайте apple</a>.
Сохраняем Makefile в директории проекта, делаем make. Если все сделано правильно - на выходе должен быть бинарник с именем vTerminal. Ура! Но нет, рано. Дело в том что запустить мы его не сможем - это гуевое приложение, и с командной строки просто так не запустится. Надо его инсталлить в систему. Разбираться в тонкостях ios на этом этапе я не стал и решил собрать простой deb-пакет что бы установить его с помощью dpkg. Итого - разбираемся в устройстве .deb и учим make собирать такой пакет.
Для этого нам надо понимать два момента - структуру .deb пакета и состав гуевого приложения под ios.
На .deb пакетах сильно растекаться не буду, кому надо подробностей - вот шикарная статья о том <a href="http://habrahabr.ru/post/78094/">как собрать deb пакет</a>. 
Что касается нашего приложения - на данном этапе обозначу control-файл пакета:

```Ini
Package: net.openios.vterminal
Name: vTerminal
Depends: bash
Version: 0.1
Architecture: iphoneos-arm
Description: A MobileTerminal fork
Maintainer: hCc <hCc@openios.net>
Author: hCc <hCc@openios.net>
Section: Utilities
```
Собственно комментировать тут особо нечего, тут и так все понятно. Package может быть любым уникальным тестовым идентификатором, в мире apple принята реверсная запись домена, но это не обязательно. В зависимостях - bash, потому как не будет шелла - некуда коннектится терминалу. dpkg кстати зависимости не обрабатывает, это делает apt, но тем не менее пусть будет, так правильнее.
В общем здесь вопросов возникнуть не должно, предлагаю переходить к структуре приложения.

Минимальное gui-ios приложение может разместиться в одной директории и содержать следующие файлы:
<ul>
<li>бинарник приложения (тот самый, который мы уже получили)</li>
<li>Info.plist - базовые настройки приложения (в том числе используемые системой при запуске приложения)</li>
<li>*.png - набор значков приложения</li>
</ul>

На картинках тоже сильно распыляться не буду, я просто скопировал все значки с директории MTerminal. Интересующиеся могут ознакомиться с <a href="https://developer.apple.com/library/ios/qa/qa1686/_index.html">требованиями и рекомендациями apple к набору значков приложения</a>.

Что касается Info.plist, это xml файл с некоторыми настройками приложения. Здесь я рассмотрю info.plist терминала, за подробностями формата можно сходить сюда <a href="https://developer.apple.com/library/ios/documentation/General/Reference/InfoPlistKeyReference/Introduction/Introduction.html">Info.plist key reference</a>.

```XML
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>CFBundleDevelopmentRegion</key>
	<string>English</string>
	<key>CFBundleDisplayName</key>
	<string>vterminal</string>
	<key>CFBundleExecutable</key>
	<string>vterminal</string>
	<key>CFBundleIconFiles</key>
	<array>
		<string>Icon</string>
		<string>Icon@2x</string>
		<string>Icon-Small</string>
		<string>Icon-Small@2x</string>
		<string>Icon-72</string>
		<string>Icon-Small-50</string>
	</array>
	<key>CFBundleIdentifier</key>
	<string>net.openios.vterminal</string>
	<key>CFBundleInfoDictionaryVersion</key>
	<string>6.0</string>
	<key>CFBundleName</key>
	<string>vterminal</string>
	<key>CFBundlePackageType</key>
	<string>APPL</string>
	<key>CFBundleSignature</key>
	<string>????</string>
	<key>CFBundleSupportedPlatforms</key>
	<array>
		<string>iPhoneOS</string>
	</array>
	<key>CFBundleVersion</key>
	<string>1.0</string>
	<key>LSRequiresIPhoneOS</key>
	<true/>
	<key>MinimumOSVersion</key>
	<string>7.0.0</string>
	<key>UIDeviceFamily</key>
	<array>
		<integer>1</integer>
		<integer>2</integer>
	</array>
	<key>UIPrerenderedIcon</key>
	<true/>
	<key>UIStatusBarHidden</key>
	<true/>
	<key>UIInterfaceOrientation</key>
	<string>UIInterfaceOrientationLandscapeLeft</string>
</dict>
</plist>
```


Ну что же, пора собирать deb пакет.

----
А десерт - небольшая утилитка по конвертации бинарных .plist в xml-ные и обратно. Пакет оформлять не стал, прикладываю сорцы:
<spoiler title="plistconvert.m 172 lines">
```objective-c
/* plistconvert: convert plist files.
   Copyright 2015
   Trusted Software Foundation.

   This file is part of openIOS utils.
   See more on openios.net
   
   This program is totally free software,
   you can do with it whatever you want.
*/

#import <Foundation/Foundation.h>
#include <stdio.h>

#define REPORT_BUGS_TO "root@openios.net"
#define VERSION_STRING "v0.3"

#define BIN 1
#define XML 0

int plistConvert(const char *, int, char *);

void print_version (const char *name)
{
  printf (" %s %s\n", name, VERSION_STRING);
  printf ("Copyright 2015 Trusted Software Foundation\n");
  printf ("licensed: as is.\n");
  exit (0);
}

static void usage (int help, const char *name)
{
  FILE *s;

  s = help ? stdout : stderr;
  
  fprintf (s, "Usage: %s -options plist_files\n", name);
  fprintf (s, " Convert *.plist files to readable xml format or back to binary.\n");
  fprintf (s, " Saves files with the same name.\n");
//  fprintf (s, " If there are no files tries to read stdin(piping)\n");
  fprintf (s, " The options are:\n\
  -b                              convert to binary format\n\
  -t -x                           convert to text(xml) format\n\
  -h -H --help                    print this help message\n\
  -v -V --version                 print version information\n");


  if (help)
    fprintf (s, "Report bugs to %s\n", REPORT_BUGS_TO);

  exit (help ? 0 : 1);
}

int main(int argc, char *argv[])
{
    if (argc < 1) usage (0, program_name);
    
    const char *program_name;
    program_name = argv[0];
    int i = 2;
    int targetFormat;
    const char *status[80];
  
    if (argc < 2) usage (0, program_name);
  
    if (argv[1][0] == '-')
    {
        if (strcmp (argv[1], "--help") == 0
             || strcmp (argv[1], "-h") == 0
             || strcmp (argv[1], "-H") == 0)
               
            usage (1, program_name);
        
        else if (strcmp (argv[1], "--version") == 0
                     || strcmp (argv[1], "-V") == 0
                     || strcmp (argv[1], "-v") == 0)
    
                 print_version (program_name);
             else
             {
                 if (strcmp (argv[1], "-b") == 0) targetFormat = BIN;
                 else if (strcmp (argv[1], "-t") == 0
                       || strcmp (argv[1], "-x") == 0) targetFormat = XML;
                      else usage(0, program_name);
                 
                 while(i < argc)
                 {       
                     printf("%s: ", argv[i]);

                     if(0 == plistConvert(argv[i], targetFormat, status))
                         printf("Ok\n");
                     else printf("failed with error: %s \n", status);
                     
                     i++;
                 }
                 exit(0);   
             }

    }
  
    usage (0, program_name);
}

plistError(NSError *error, char *status)
{
    char *cError;
    char *hint = "";
    char *notFound = "(file not found?)";
    char *notPlist = "(file not *.plist?)";
    char *chmod = "(can't write, not enough rights?)";
    int code;
  
    cError = [[error domain] UTF8String];
    code = [error code];
    
    if(strcmp (cError, "NSCocoaErrorDomain") == 0)
    {
        switch (code)
        {
            case 260:  hint = notFound; break;
            case 3840: hint = notPlist; break;
            case 513:  hint = chmod;    break;
        }
    }

    snprintf(status, 80, "%s: code %d, %s", cError, code, hint);
    
    return 1;

}

int plistConvert(const char *fileName, int targetFormat, char *status)
{
    NSError *error;
    NSPropertyListFormat format;
  
    NSString *path = [NSString stringWithCString: fileName
                               encoding: NSASCIIStringEncoding];
                      
    NSData *plistData = [NSData dataWithContentsOfFile:path
                                options:NSDataReadingUncached
                                error:&error];
  
    if(error != NULL) return plistError(error, status);
  
    id plist = [NSPropertyListSerialization propertyListWithData:plistData
                    options:NSPropertyListImmutable
                    format:&format
                    error:&error];

    if(error != NULL) return plistError(error, status);

    NSString *xmlData = [NSPropertyListSerialization dataWithPropertyList:plist
                            format:(targetFormat==XML?
                                        NSPropertyListXMLFormat_v1_0:
                                        NSPropertyListBinaryFormat_v1_0)
                            options:0
                            error:&error];
    if(error == NULL)
    {
        [xmlData writeToFile:path
                 options:NSDataWritingAtomic
                 error:&error];

        if(error != NULL) return plistError(error, status);

        return 0;
    }
    else return plistError(error, status);

}
```
</spoiler>
собираем простой строкой:

```bash
clang -v plistconvert.m -o plistconvert \
    -Xlinker -L/User/SDKs/7.1/usr/lib/ \
    -Xlinker -L/User/SDKs/7.1/usr/lib/system/ \
    -isysroot /User/SDKs/7.1/  \
    -ObjC -framework Foundation
```
и пользуемся. И комплимент от шеф-повара всем, осилившим первое, второе и десерт. Многие настройки ios хранятся в plist листах, так что мы можем тюнить систему, простой последовательностью:
plistconvert -x *.plist
vim *.plist
plistconvert -b *.plist

делаем приложение неудаляемым (у кого дома дети - поймут зачем), а неудаляемые - удаляем (поймут те, кому нафиг не сдался киоск):


делаем приложение невидимым(находится через спот
