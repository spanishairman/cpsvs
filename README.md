#### КриптоПро SVS. Обновление списка отозванных сертификатов УЦ ФНС России в локальном хранилище сертификатов
В соответствии с Федеральным законом от 06.04.2011 г. № 63-ФЗ, с 1 января 2022 года на ФНС России возлагаются функции по выпуску квалифицированной электронной подписи юридических лиц (лиц, имеющих право действовать от имени юридического лица без доверенности), индивидуальных предпринимателей и нотариусов. 

Актуальная информация о корневых сертификатах и опубликованных списках отзыва публикуется на странице "Ресурсы Удостоверяющего центра ФНС России".

Адрес [страницы](https://www.nalog.gov.ru/rn77/related_activities/ucfns/ccenter_res/)

Прилагаемый [Powershell скрипт](crldownload.ps1) закачивает, если не были закачаны ранее, и устанавливает в локальное хранилище SVS-Ca актуальные списки отзыва сертификатов УЦ ФНС России. В случае недоступности точек распространения списков аннулированных сертификатов, на электронную почту ответственного лица приходит сообщение об ошибке.

Безотносительно программного обеспечения КриптоПро SVS, скрипт будет полезен при  проверке электронных подписей в любом ПО, если изменить значение переменной $svsstore, отвечающей за параметр хранилище сертификатов, с SVS-Ca на CA.

Для уже скачанных на момент запуска скрипта списков отзыва вычисляются их контрольные суммы и сравниваются с контрольными суммами файлов crl, опубликованных на удаленных веб-серверах. В случае различия полученных значений контрольных сумм, закачивается и устанавливается актуальная версия CRL, при совпадении значений - список пропускается. Данный скрипт можно запускать с помощью планировщика заданий, что позволит держать актуальной информацию об отзыве сертификатов ЭП, сократить время проверки и избежать ошибок, вызванных сетевыми задержками.
```
# Ресурсы Удостоверяющего центра ФНС России
# https://www.nalog.gov.ru/rn77/related_activities/ucfns/ccenter_res/
# Сертификаты УЦ ФНС
# http://pki.tax.gov.ru/crt/CA_FNS_Russia_2019_UL.crt
# http://pki.tax.gov.ru/crt/CA_FNS_Russia_2022_01.crt
# http://pki.tax.gov.ru/crt/CA_FNS_Russia_2022_02.crt
# http://cdp.tax.gov.ru/crt/CA_FNS_Russia_2023_01.crt
# Списки отозванных сертификатов УЦ ФНС
# http://pki.tax.gov.ru/cdp/4e5c543b70fefd74c7597304f2cacad7967078e4.crl
# http://pki.tax.gov.ru/cdp/fcb21945f2bb7670b371b03cee94381d4f975cd5.crl
# http://pki.tax.gov.ru/cdp/e91f07442c45b2cf599ee949e5d83e8382b94a50.crl
# http://cdp.tax.gov.ru/cdp/d156fb382c4c55ad7eb3ae0ac66749577f87e116.crl
# Сертификаты УЦ технологического УЦ ФНС
# http://uc.nalog.ru/crt/CA_FNS_Russia_2018.crt
# http://uc.nalog.ru/crt/CA_FNS_Russia_2022.crt
# Списки отзыва Технологического удостоверяющего центра
# http://uc.nalog.ru/cdp/ac53bead76ac54d0880675d705c58b01b5abbe94.crl
# http://uc.nalog.ru/cdp/c1836f3194b61e57ba10a847870a51e399cb07d0.crl 
# ---------------------------------------------------------------
# Переключаем кодировку в UTF-8.
# Переключение кодировок не помогает от крякозябров на кириллице, которые пишутся в журнал.
# Проверял на кодировках unicode, utf8, unknown, default, ascii, oem, string, utf32. Не стал проверять bigendianunicode, utf7.
# $OutputEncoding = [Console]::InputEncoding = [Console]::OutputEncoding = New-Object System.Text.UTF8Encoding
# $PSDefaultParameterValues['Out-File:Encoding'] = 'utf8'
$scriptstart = $( Get-Date )
$PSEmailServer = "smtphost.example.ru"
$msndr = "svsadmin@example.ru"
$mrcpt = "admin@example.ru"
cd $PSScriptRoot
if ( -not ( Test-Path -PathType Container -Path ./Crl ) )
 {
     Write-Output "Создаём каталог для списков отзыва"
  New-Item -Path . -Name "Crl" -ItemType "directory"
 }
if ( -not ( Test-Path -PathType Container -Path ./Log ) )
 {
    Write-Output "Создаём каталог для файлов журнала"
    New-Item -Path . -Name "Log" -ItemType "directory"
 }
$logfile = "./Log/GetCRLs-$( Get-Date -Format "yyyy-MM-dd_HH-mm" ).log"
Write-Output "Скрипт запущен $scriptstart" > $logfile
Write-Output "Удаляем файлы журналов старше 30 дней" >> $logfile
Get-ChildItem "./Log" -Recurse -Filter *.log -File | Where LastWriteTime -lt  (Get-Date).AddDays(-30) | Remove-Item -Force | Write-Output >> $logfile
$wc = [System.Net.WebClient]::new()
$svsstore = "SVS-Ca"
$urls = get-content "./urls.info"
Write-Output "Считываем адреса списков отзывов из urls.info" >> $logfile
ForEach ($url in $urls)
 {
        Write-Output "Адрес загрузки списка отзыва - $url" >> $logfile
 $crl = Split-Path -Path "$url" -Leaf
 Write-output "Файл списка отзыва - $crl" >> $logfile
 if ( -not ( Test-Path -Path ./Crl/$crl ) )
  {
  Write-Output "Локальная версия списка $crl отсутствует. закачиваем с удаленного сервера" >> $logfile
  $Error.clear()
  wget $url -OutFile ./Crl/$crl | Write-Output >> $logfile
   if ( -not $Error )
    {
    Write-Output "Устанавливаем $crl в хранилище $svsstore" >> $logfile
    certutil.exe -addstore -f $svsstore ./Crl/$crl | Write-Output >> $logfile
    }
    else {
         Send-MailMessage -From $msndr -To $mrcpt -Subject "$env:COMPUTERNAME. Error loading CRL $crl" -Body "Error loadng CRL $crl from source $url on $env:COMPUTERNAME. Error: $Error"
         }
  }
  $crlfilehash = Get-FileHash ./Crl/$crl
  Write-Output "Контрольная сумма скачанного списка $crl :" $crlfilehash.Hash >> $logfile
  $crlurlhash = Get-FileHash -InputStream ($wc.OpenRead($url))
  if ( $crlurlhash.Hash -eq $null )
   {
   Write-Output "Ошибка доступа к серверу публикации CRL $url" >> $logfile
   Send-MailMessage -From $msndr -To $mrcpt -Subject "$env:COMPUTERNAME. Error get Filehash for CRL $crl" -Body "Error get Filehash for CRL $crl from source $url on $env:COMPUTERNAME. Error: $Error"
   }
  Write-Output "Контрольная сумма списка $crl на удаленном сервере :" $crlurlhash.Hash >> $logfile
  if ( ( $crlurlhash.Hash -ne $crlfilehash.Hash ) -and ( $crlurlhash.Hash -ne $null ) )
   {
   Write-Output "Так как отпечатки для $crl на локальном компьютере и на удаленном сервере отличаются, то скачиваем актуальную версию" >> $logfile
   $Error.clear()
   wget $url -OutFile ./Crl/$crl | Write-Output >> $logfile
    if ( -not $Error )
    {
    Write-Output "Устанавливаем скачанный $crl в хранилище $svsstore" >> $logfile
    certutil.exe -addstore -f $svsstore ./Crl/$crl | Write-Output >> $logfile
    }
    else {
         Send-MailMessage -From $msndr -To $mrcpt -Subject "$env:COMPUTERNAME. Error loading CRL $crl" -Body "Error loadng CRL $crl from source $url on $env:COMPUTERNAME. Error: $Error"
         }
   }
 }
$scriptend = $( Get-Date )
Write-Output "Скрипт завершил работу $scriptend" >> $logfile
$timespan = New-TimeSpan -Start $scriptstart -End $scriptend
Write-Output "Время выполнения скрипта составило $timespan" >> $logfile
```
Для работы скрипта потребуется файл, в данном случае [urls.info](urls.info), в котором построчно перечислены все адреса публикаций списков аннулированных сертификатов, которые необходимо загрузить и установить.
```
http://pki.tax.gov.ru/cdp/4e5c543b70fefd74c7597304f2cacad7967078e4.crl
http://pki.tax.gov.ru/cdp/fcb21945f2bb7670b371b03cee94381d4f975cd5.crl
http://pki.tax.gov.ru/cdp/e91f07442c45b2cf599ee949e5d83e8382b94a50.crl
http://cdp.tax.gov.ru/cdp/d156fb382c4c55ad7eb3ae0ac66749577f87e116.crl
http://uc.nalog.ru/cdp/ac53bead76ac54d0880675d705c58b01b5abbe94.crl
http://uc.nalog.ru/cdp/c1836f3194b61e57ba10a847870a51e399cb07d0.crl
```
Журналы работы скрипта хранятся во вложенном каталоге Log. Подкаталоги Crl и Log создаются автоматически при первом запуске.
