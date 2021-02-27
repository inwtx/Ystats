```
#!/bin/bash
#
# Ystats v2.0
#
# Script to build server statistics Ystats.html
#
#------------------------------------------------------------------------#
#              --- Server statistics web page builder ---                #
#                                                                        #
# The following script can be used to monitor a YAMN rmailer servers.    #
# Several statistics concerning the server and YAMN are displayed.       #
# It must be run once a minute.                                          #
#                                                                        #
# Retrieve stastics for server web page - cron job                       #
# */1 * * * * /path/to/Ystats.sh                                         #
#                                                                        #
#------------------------------------------------------------------------#

#top
export PATH=$PATH:/sbin
export PATH=$PATH:/bin

myemailaddr=""     # your private email address
webpgnm="/Ystats.html"       # webpage name
yamnpath="/home/yamn/yamn"           # no trailing /
webpgpath="/var/www/html"            # path to server webpage folder
fileoffset="y"                       # unique internal file names ID
sshlog="/var/log/auth.log"
certpath="/etc/ssl/private/letsencrypt-domain.pem"
tempdisp="yes"
dostats="y"
expdwarn=30
filePath=${0%/*}  # current file path
stathighlight="#ff1493"
varupt=$(uptime)
mmid=$remailerid
tempvar=""
tempnum=0
fontsz="2"
hfontsz="2"
bgclr=#EBEADF
fontcolor=#000040
titlecolor=#f0f0f0
fontcl=#FFFFFF
machinewidth=430
roguewidth=200
freewidth=230
netstatswidth=230
mixmasterwidth=200
iptableswidth=125
miscstats=200
mixfiles=160
remailerstatistics=230
titlecolor=#990066
MachineRogueTableWidth=1112
MixMiscRemailerTableWidth=870

statarray=(
"http://www.mixmin.net/yamn/mlist.txt;mixmin4096;mixmin"  # mlist url;title
"https://cloaked.pw/yamn/mlist.txt;cloaked"
)

function poolcount(){      #count pool for day
        if [ ! -s $filePath/savepool.$fileoffset.txt ];  then                             # create file first time
            echo "0" > $filePath/savetodaypoolcnt.$fileoffset.txt
            echo "0" >> $filePath/savetodaypoolcnt.$fileoffset.txt
            echo "" > $filePath/savepool.$fileoffset.txt
            else
            if [ $(date +"%H:%M") = "00:00" ]; then  # reset at midnight
               savetodaypoolcnt=$(head -n 1 $filePath/savetodaypoolcnt.$fileoffset.txt)  # save previous days count
               echo "0" > $filePath/savetodaypoolcnt.$fileoffset.txt                      # zero new days count
               echo "$savetodaypoolcnt" >> $filePath/savetodaypoolcnt.$fileoffset.txt    # save previous days count
               Tsavetodaypoolcnt=0                                             # zero out todays bucket
               echo "$(ls $yamnpath/pool/)" > $filePath/savepool.$fileoffset.txt           # save current pool at BOD for comparison
            else
               savetodaypoolcnt=$(head -n 1 $filePath/savetodaypoolcnt.$fileoffset.txt)
               savepriorpoolcnt=$(sed -n 2p $filePath/savetodaypoolcnt.$fileoffset.txt)
               echo "$(ls $yamnpath/pool/)" > $filePath/temppool.$fileoffset.txt           # get current pool in seq list

               while read line ; do
                  if grep $line $filePath/savepool.$fileoffset.txt ; then                 # is this message in Tsavepool?
                     continue                                                  # yes
                     else
                    ((savetodaypoolcnt++))                                    # found a new message
                    echo $line >> $filePath/savepool.$fileoffset.txt                      # Tsave the message file name for future ref
                  fi
               done<$filePath/temppool.$fileoffset.txt                                    # read file line by line
               echo $savetodaypoolcnt > $filePath/savetodaypoolcnt.$fileoffset.txt       # save the pool cnt for next chk
               echo $savepriorpoolcnt >> $filePath/savetodaypoolcnt.$fileoffset.txt      # save the pool cnt for next chk

               rm $filePath/temppool.$fileoffset.txt
            fi
        fi
}


cat /dev/null > $webpgpath/$webpgnm  # clear html file

echo "<html><head><title>Server Stats</title></head><body bgcolor=\"$bgclr\" TEXT=\"$fontcolor\" LANG=\"en-US\" DIR=\"LTR\">" > $webpgpath/$webpgnm

##'-------------------'
## BEGIN Top date line
##'-------------------'
echo "<font face=\"Verdana\" size=\"=\"$fontsz\"\" color=\"$fontcolor\"><b>&nbsp;" >> $webpgpath/$webpgnm
MLvar=$(date | awk '{print $0" "$2" "$3" "$6" "$4}' | awk '{print "("$1") "$7" "$8", "$9" &nbsp;&nbsp; "$10}')
MLvar="${MLvar%:*} $(date | awk '{print $5}') &nbsp;&nbsp; ${0##*/}"
echo "&nbsp;&nbsp;$MLvar" >> $webpgpath/$webpgnm
echo "</font></b><br>" >> $webpgpath/$webpgnm
##'-------------------'
##  END Top date line
##'-------------------'


##'----------------------------------------------'
##'----------------------------------------------'
## BEGIN Machine & Netstats/Free horzontal tables
##'----------------------------------------------'
##'----------------------------------------------'
echo "<table><tr valign=\"top\"><td>" >> $webpgpath/$webpgnm


##'-------------------'
## BEGIN Machine table
##'-------------------'
#echo "<table border=\"1\" cellpadding=\"2\" cellspacing=\"0\"><tr><td width=\"$machinewidth\" bgcolor=\"$titlecolor\">
echo "<table border=\"1\" cellpadding=\"2\" cellspacing=\"0\"><tr><td bgcolor=\"$titlecolor\">
<b><font face=\"Verdana\" color=\"$fontcl\" size=\"$fontsz\"><b>Machine</b></font></td></tr>
<tr><td><font face=\"Courier New\" size=\"$fontsz\" color=\"$fontcolor\"><b>" >> $webpgpath/$webpgnm

## linux info
echo "$(lsb_release -d) $(uname -m)<br>" | sed -e 's/Description://g' | tr -d "\t" >> $webpgpath/$webpgnm

# ip address
echo "IP server: $(hostname -I)<br>" >> $webpgpath/$webpgnm

# domain name
echo "Domain name: $(dig -x $(hostname -I) +short | awk -F '.' '{print $1"."$2}')<br>" >> $webpgpath/$webpgnm

# MTU
echo "$(ifconfig | sed '1!d' | awk '{print "MTU: " $4}')<br>" >> $webpgpath/$webpgnm

## ipv4 TTL & ipv6 HOP count
echo "ipv4 TTL: $(sysctl net.ipv4.ip_default_ttl | awk '{ print $3 }')<br>" >> $webpgpath/$webpgnm
#echo "ipv6 HOP: $(sysctl net.ipv6.conf.all.hop_limit | awk '{ print $3 }')<br>" >> $webpgpath/$webpgnm

## openssl
#vartemp=$(openssl version | cut -d'L' -f2)
#echo -n "OpenSSL: $vartemp<br>" >> $webpgpath/$webpgnm
echo "$(openssl version)" | awk '{print $1" "$2" ("$3" "$4" "$5")<br>"}' >> $webpgpath/$webpgnm

## certificate expdt
#notAfter=Feb  8 12:19:20 2021 GMT
SSLvar=$(openssl x509 -enddate -noout -in $certpath)  # /etc/ssl/private/letsencrypt-domain.pem)
SSLvar=$(cut -d'=' -f2 <<<$SSLvar)
SSLvar=$(awk '{print $1"-"$2"-"$4" "$3" "$5}' <<<$SSLvar)
SSLvar1=$(awk '{print $1}' <<<$SSLvar)
SSLvar2=$(let DIFF=(`date +%s -d $SSLvar1` - `date +%s`)/86400 && echo $DIFF)
if [[ $SSLvar2 -le 3 ]];then
   echo "<font color=FF0000><b>Cert expdt: $SSLvar<br></b></font>" >> $webpgpath/$webpgnm
   else
   echo "Cert expdt: $SSLvar<br>" >> $webpgpath/$webpgnm
fi

## last boot time
varbt=`echo $(who -b)`
echo ${varbt/system boot/Last system boot:} >> $webpgpath/$webpgnm

## up time
varupt=`echo ${varupt//up/Up}`
varupt=`echo ${varupt//load/Load}`
awk -F '[ \t\n\v\r]' '{print "<br>"$2" "$3" "$4" "$5" "$8" "$9" "$10" "$11" "$12" "$13" "$14" "$15}' <<< $varupt >> $webpgpath/$webpgnm

###Bandwidth
  ###Month to date bandwidth
if pidof -x "vnstatd" >/dev/null; then
   var1=$(date |  awk '{print $2" "}')
   var2=$(date +"'%y")
   var3=$var1$var2
   var4=$(grep -e "$var3" <<< $(vnstat -m))
   bline=$(awk '{print "MTD bandwidth: " $1" "$2" - "$9" "$10}' <<<$var4)
   echo "<br>$bline" >> $webpgpath/$webpgnm
else
   echo "<br>MTD bandwidth: vnstat not running!" >> $webpgpath/$webpgnm
fi

## last login
(last -a root) > $filePath/templ.$fileoffset.txt
sed -n -e 1p $filePath/templ.$fileoffset.txt > $filePath/templ2.$fileoffset.txt
(cat $filePath/templ2.$fileoffset.txt | colrm 1 22 | colrm 35 100) > $filePath/templ.$fileoffset.txt
lines=$(head -n 1 $filePath/templ.$fileoffset.txt)
echo "<br>Last login: $lines" >> $webpgpath/$webpgnm
echo "</font></b></td></tr></table><br>" >> $webpgpath/$webpgnm    # end build Machine
##'-----------------'
## END Machine table
##'-----------------'


##'------------------------------------'
echo "</td><td>" >> $webpgpath/$webpgnm  # MIDDLE: vertical divider for Machine and Netstat/Free
##'------------------------------------'


##'-------------------'
## BEGIN netstat table
##'-------------------'
echo "<table border=\"1\" cellpadding=\"2\" cellspacing=\"0\"><tr><td bgcolor=\"$titlecolor\">
<font face=\"Verdana\" color=\"$fontcl\" size=\"$fontsz\"><b>Netstats</b></font></td></tr>
<tr><td><font face=\"Courier New\" size=\"$fontsz\" color=\"$fontcolor\"><b>" >> $webpgpath/$webpgnm
netstat -vatnp > $filePath/templ.$fileoffset.txt
sed -i "/tcp6/d" $filePath/templ.$fileoffset.txt            # remove tcp6 lines
sed -i "/python2.7/d" $filePath/templ.$fileoffset.txt       # remove bitmessage python2.7 messages
sed -i "/nginx\: worker/d" $filePath/templ.$fileoffset.txt  # remove nginx: worker messages
sed -i "/TIME_WAIT/d" $filePath/templ.$fileoffset.txt       # remove TIME_WAIT messages
sed -i 's/ /\&nbsp;/g' $filePath/templ.$fileoffset.txt
sed -i 's/ //g' $filePath/templ.$fileoffset.txt
sed -e 's/$/<br>/' $filePath/templ.$fileoffset.txt >> $webpgpath/$webpgnm
echo "</font></td></tr></table><br>" >> $webpgpath/$webpgnm
##'-----------------'
## End netstat table
##'-----------------'


## no NIDDLE divider needed - tables are vertically inclined


##'-----------------'
## BEGIN Free  table
##'-----------------'
# free begin
echo "<table border=\"1\" cellpadding=\"2\" cellspacing=\"0\"><tr><td bgcolor=\"$titlecolor\">
<font face=\"Verdana\" color=\"$fontcl\" size=\"$fontsz\"><b>Free</b></font></td></tr>
<tr><td><font face=\"Courier New\" size=\"$fontsz\" color=\"$fontcolor\"><b>" >> $webpgpath/$webpgnm

free > $filePath/templ.$fileoffset.txt
sed -i 's/ /\&nbsp;/g' $filePath/templ.$fileoffset.txt
sed -i 's/ //g' $filePath/templ.$fileoffset.txt
sed -e 's/$/\&nbsp;\&nbsp;<br>/' $filePath/templ.$fileoffset.txt >> $webpgpath/$webpgnm  # append <br>
echo "</font></td></tr></table><br>" >> $webpgpath/$webpgnm
##'--------------'
## END Free table
##'--------------'


##'-------------------------------------------'
##'-------------------------------------------'
## END Machine & Netstat/Free horzontal tables
##'-------------------------------------------'
##'-------------------------------------------'
echo "</td></tr></table>" >> $webpgpath/$webpgnm


##'---------------------------------------------------------'
##'---------------------------------------------------------'
## BEGIN Misc Stats, YAMN stats, & YAMN IDs horzontal tables
##'---------------------------------------------------------'
##'---------------------------------------------------------'
echo "<table><tr valign=\"top\"><td>" >> $webpgpath/$webpgnm  # BEGIN divider table for Mix Files, Misc Stats, and Remailer stats


##'-----------------------'
## BEGIN Misc Stats  table
##'-----------------------'

echo "<table border=\"1\" cellpadding=\"2\" cellspacing=\"0\"><tr><td bgcolor=\"$titlecolor\">
<font face=\"Verdana\" color=\"$fontcl\" size=\"$fontsz\"><b>Misc Stats</b></font></td></tr>
<tr><td><font face=\"Courier New\" size=\"$fontsz\" color=\"$fontcolor\"><b>" >> $webpgpath/$webpgnm

## /var/log size begin
#Size /var/log
tempvar=$(du -sh /var/log)
tempvar=`echo "Size /var/log: " && echo ${tempvar//\/var\/log} && echo "<br>"`
echo $tempvar >> $webpgpath/$webpgnm

#Website hits
tempvar2=$(cat /var/log/nginx/access.log | sort | awk '{print $1}' | grep -v 127.0.0.1 | grep -v 50.26.242.136 | grep -v  23.237.26.103 | wc -l)
tempvar=`echo "Website:&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;" && \
echo $(cat /var/log/nginx/access.log | sort | awk '{print $1}' | grep -v 127.0.0.1 | grep -v 50.26.242.136 | grep -v  23.237.26.103 | wc -l) && echo "<br>"`

if [[ $tempvar2 -gt 50 ]]; then
   echo "<font color=FF0000><b>"$tempvar"</b></font>" >> $webpgpath/$webpgnm  # color 'Website' hits red
   else
echo $tempvar >> $webpgpath/$webpgnm
fi

#mail log rejects
tempvar=`echo "Rejected mail: " && echo $(grep -c reject /var/log/mail.log) && echo "<br>"`

if [[ $(grep -c reject /var/log/mail.log) -gt 0 ]]; then
   echo "<font color=FF0000><b>"$tempvar"</b></font>" >> $webpgpath/$webpgnm  # color 'Rejected mail:' red
   else
   echo $tempvar >> $webpgpath/$webpgnm
fi

#Hits
if [ -e $filePath/AccessLogHits.$fileoffset.txt ] && [[ $(cat $filePath/AccessLogHits.$fileoffset.txt | wc -l) -gt 0 ]]; then
   tempvar3=$(wc -l $filePath/AccessLogHits.$fileoffset.txt | awk '{print $1}')
   tempvar=`echo "<font color=FF0000><b>Hits:&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;$tempvar3</b></font>" && echo "<br>"`
   echo $tempvar >> $webpgpath/$webpgnm
fi
echo "</font></td></tr></table><br>" >> $webpgpath/$webpgnm   # end build Misc
##'--------------------'
## END Misc Stats table
##'--------------------'


##'------------------------------------'
echo "</td><td>" >> $webpgpath/$webpgnm  # MIDDLE: vertical divider for Misc Stats & YAMN Stats
##'------------------------------------'


##'----------------------'
## BEGIN YAMN Stats table
##'----------------------'
echo "<table border=\"1\" cellpadding=\"2\" cellspacing=\"0\"><tr><td bgcolor=\"$titlecolor\">
<font face=\"Verdana\" color=\"$fontcl\" size=\"$fontsz\"><b>YAMN Stats</b></font></td></tr>
<tr><td><font face=\"Courier New\" size=\"$fontsz\" color=\"$fontcolor\"><b>" >> $webpgpath/$webpgnm

## mailq
echo "mailq count: " > $filePath/templ.$fileoffset.txt
vattest=$(mailq)
if [[ $vattest = "Mail queue is empty" ]]; then
   cat /dev/null > $filePath/notification$fileoffset.txt
   echo "0" >> $filePath/templ.$fileoffset.txt
   varzero="y"
   else
   varzero="n"
   mailq | grep -c "^[A-Z0-9]" >> $filePath/templ.$fileoffset.txt
   varmq=$(mailq | grep -c "^[A-Z0-9]")

   if [ $(expr $(date +"%H") % 4) -eq 0 -a $(date +"%M") = "00" -a $varmq -gt 200 ]; then   # mail every 4 hours if mailq > 9
      mutt -s "Error from $myemailaddr - Mailq backup - $varmq" $myemailaddr <<<"From /etc/Servstats/Lstats.sh: Mailq backup - $varmq"
   fi
fi

sed '1~2 {N;N;s/\n/ /g}' $filePath/templ.$fileoffset.txt > $filePath/templ2.$fileoffset.txt  # concat lines 1 and 2
if [[ $varzero = "n" ]]; then
   vartemp=$(<$filePath/templ2.$fileoffset.txt)
   echo "<font color=FF0000><b>"$vartemp"</b></font>" > $filePath/templ2.$fileoffset.txt  # color 'mailq count:' red
fi

cat $filePath/templ2.$fileoffset.txt >> $webpgpath/$webpgnm
echo "<br>" >> $webpgpath/$webpgnm
## mailq end

## pool count
poolcount

echo "pool count:&nbsp;" > $filePath/templ.$fileoffset.txt
find /home/yamn/yamn/pool -type f | wc -l >> $filePath/templ.$fileoffset.txt
sed '1~2 {N;N;s/\n/ /g}' $filePath/templ.$fileoffset.txt > $filePath/templ2.$fileoffset.txt  # concat lines 1 and 2
cat $filePath/templ2.$fileoffset.txt >> $webpgpath/$webpgnm
echo "<br>" >> $webpgpath/$webpgnm
## pool count end

## pool today and yesterday
if [ ! -s $filePath/savetodaypoolcnt.$fileoffset.txt ]; then
   ptd=0
   pyd=0
   echo "0" > $filePath/savetodaypoolcnt.$fileoffset.txt
   echo "0" >> $filePath/savetodaypoolcnt.$fileoffset.txt
fi

if [[ $(cat $filePath/savetodaypoolcnt.$fileoffset.txt | wc -l) -gt 0 ]]; then  # running poolcount.sh?
   ptd=$(head -n 1 $filePath/savetodaypoolcnt.$fileoffset.txt)  # get 1st line = total thus far today
   pyd=$(sed -n 2p $filePath/savetodaypoolcnt.$fileoffset.txt)  # get 2nd line = prior
   echo "pool today: &nbsp;$ptd<br>" >> $webpgpath/$webpgnm
   echo "pool prior: &nbsp;$pyd<br>" >> $webpgpath/$webpgnm
fi
## pool today and yesterday end

echo "</font></td></tr></table><br>" >> $webpgpath/$webpgnm   # end build Misc
##'--------------------'
## END YAMN Stats table
##'--------------------'


##'------------------------------------'
echo "</td><td>" >> $webpgpath/$webpgnm  # MIDDLE: vertical divider for YAMN Stats & YAMN
##'------------------------------------'


##'----------------'
## BEGIN YAMN table
##'----------------'
echo "<table border=\"1\" cellpadding=\"2\" cellspacing=\"0\"><tr><td bgcolor=\"$titlecolor\">
<font face=\"Verdana\" color=\"$fontcl\" size=\"$fontsz\"><b>YAMN</b></font></td></tr>
<tr><td><font face=\"Courier New\" size=\"$fontsz\" color=\"$fontcolor\"><b>" >> $webpgpath/$webpgnm

## top stats
top -b -n1 > $filePath/templ.$fileoffset.txt             # get TOP listing of exes
var1=$(grep 'PID USER' $filePath/templ.$fileoffset.txt)  # get header line
var2=$(grep -w yamn $filePath/templ.$fileoffset.txt)        # get yamn line
echo "$var1" > $filePath/templ.$fileoffset.txt
echo "$var2" >> $filePath/templ.$fileoffset.txt
sed -i 's/ /\&nbsp;/g' $filePath/templ.$fileoffset.txt
sed -e 's/$/<br>/' $filePath/templ.$fileoffset.txt >> $webpgpath/$webpgnm  # append <br>
echo "<br>" >> $webpgpath/$webpgnm

## start date/time
echo $(grep "yamn" <<< "$(ps -eo lstart,cmd)") | awk '{print "Started: "$1" "$2" "$3" "$4}' | cut -c 1-28 >> $webpgpath/$webpgnm

## md5
echo "<br>MD5: $(md5sum $yamnpath/yamn)" > $filePath/templ.$fileoffset.txt
cat $filePath/templ.$fileoffset.txt >> $webpgpath/$webpgnm

## pub key
# var1=$(head -n 1 /home/yamn/yamn/key.$fileoffset.txt)   # get pubkey header
# awk '{print "<br>Pubkey: " $3}' <<< "$var1" >> $webpgpath/$webpgnm

## key(s)
var1="<br>Pub/Sec key: "
grep "Created:" $yamnpath/secring.mix > $filePath/temp1.$fileoffset.txt  # extract Created: date
grep "Expires:" $yamnpath/secring.mix > $filePath/temp12.$fileoffset.txt  # extract Expires: date

cat $filePath/temp12.$fileoffset.txt > $filePath/temp16.$fileoffset.txt
sed -i 's/Expires: //g' $filePath/temp16.$fileoffset.txt  # del Expires:

paste $filePath/temp1.$fileoffset.txt $filePath/temp12.$fileoffset.txt > $filePath/temp13.$fileoffset.txt  # stack dates side by side
sed -i 's/\t/ /g' $filePath/temp13.$fileoffset.txt              # replace tab btwn dates with space
sed -i 's/Created: //g' $filePath/temp13.$fileoffset.txt
sed -i 's/Expires: //g' $filePath/temp13.$fileoffset.txt
var7=$(awk 'c&&!--c;/Expires:/{c=1'} $yamnpath/secring.mix)
echo "$var7" > $filePath/temp14.$fileoffset.txt
var2=0

while read line1; do
   ((var2++))
   var3=$(awk -F '[ \t\n\v\r.]' '{print $1" "$2}' <<< $line1)  # get 2014-05-08 2014-11-04
   var9=$(awk -v var8=$var2 'NR==var8' $filePath/temp16.$fileoffset.txt)  # pull stacked dates from temp12.$fileoffset.txt
   var10=$(date +"%Y-%m-%d")
   days=$(( ($(date --date=$var9 +%s) - $(date --date=$var10 +%s) )/(60*60*24) ))   # calc days between
   var5=$days
   var7=$(sed -n $var2'p' $filePath/temp14.$fileoffset.txt)
   var4="$var1 $var3 ($var5) ($var7)"                    #<- days remaining calc
   var4="$var1 $var3 ($var7)"                    #<- days remaining calc
   echo $var4 >> $webpgpath/$webpgnm
done<$filePath/temp13.$fileoffset.txt

echo "</font></td></tr></table><br>" >> $webpgpath/$webpgnm
##'--------------'
## END YAMN table
##'--------------'


##'-------------------------------------------------------'
##'-------------------------------------------------------'
## END Misc Stats, YAMN stats, & YAMN IDs horzontal tables
##'-------------------------------------------------------'
##'-------------------------------------------------------'
echo "</td></tr></table>" >> $webpgpath/$webpgnm


##'--------------------------------------'
##'--------------------------------------'
## BEGIN horzontal YAMN Statistics tables
##'--------------------------------------'
##'--------------------------------------'
echo "<table><tr valign=\"top\"><td>" >> $webpgpath/$webpgnm

varaYS=0

for i in "${statarray[@]}"; do
   varYS=$i
   ((varaYS++))

   echo "<table border=\"1\" cellpadding=\"2\" cellspacing=\"0\"><tr><td bgcolor=\"$titlecolor\">
   <font face=\"Verdana\" color=\"$fontcl\" size=\"$fontsz\"><b>Remailer Statistics (${varYS##*;})</b></font></td></tr>
   <tr><td><font face=\"Courier New\" size=\"$fontsz\" color=\"$fontcolor\"><b>" >> $webpgpath/$webpgnm

   if [[ $varaYS -eq 1 ]]; then  #  only pause at 1st stat download
      sleep 5                    #  pause on 1st stat collect for pingers to finish updating their stats
   fi

   wget  --no-check-certificate --timeout=15  -t 1 ${varYS%%;*} -O $filePath/varmlist.$fileoffset.txt
   echo $(date) > $filePath/statdate.$fileoffset.txt

   savdate=$(< $filePath/statdate.$fileoffset.txt)
   grep "%"  $filePath/varmlist.$fileoffset.txt | colrm 14 27 > $filePath/astats.$fileoffset.txt
   sed -i 's/\?/ /g' $filePath/astats.$fileoffset.txt
   sed -i 's/ *$//' $filePath/astats.$fileoffset.txt
   sed -i 's/ /\&nbsp;/g' $filePath/astats.$fileoffset.txt
   sed -i 's/^/\&nbsp;/' $filePath/astats.$fileoffset.txt       # prepend a blank
   sed -i 's/$/\&nbsp;/' $filePath/astats.$fileoffset.txt       # append a blank
   sed -i "1i&nbsp;$savdate" $filePath/astats.$fileoffset.txt
   sed -i 's/$/<br>/' $filePath/astats.$fileoffset.txt

#  Surround

   rm $filePath/templ.$fileoffset.txt
   cat $filePath/astats.$fileoffset.txt >> $webpgpath/$webpgnm
   echo "</font></td></tr></table>" >> $webpgpath/$webpgnm

echo "</td><td>" >> $webpgpath/$webpgnm

done

if false; then                            
##'------------------------------------'
echo "</td><td>" >> $webpgpath/$webpgnm  # MIDDLE: vertical divider for source III and MY_SYN_DROP
##'------------------------------------'


##'-----------------------'
## BEGIN MY_SYN_DROP table
##'-----------------------'
echo "<table border=\"1\" cellpadding=\"2\" cellspacing=\"0\"><tr><td bgcolor=\"$titlecolor\">
<font face=\"Verdana\" color=\"$fontcl\" size=\"$fontsz\"><b>MY_SYN_DROP</b></font></td></tr>
<tr><td><font face=\"Courier New\" size=\"$fontsz\" color=\"$fontcolor\"><b>" >> $webpgpath/$webpgnm

echo "$(iptables -L MY_SYN_DROP -n)" > $filePath/templ.$fileoffset.txt
sed -i 's/$/<br>/' $filePath/templ.$fileoffset.txt   # add <br> to end of every rec
sed -i '1,2d' $filePath/templ.$fileoffset.txt        # delete lines 1 & 2
awk '{ print $4 }' $filePath/templ.$fileoffset.txt > $filePath/temp2.$fileoffset.txt
sed -i 's/$/<br>/g' $filePath/temp2.$fileoffset.txt
cat $filePath/temp2.$fileoffset.txt >> $webpgpath/$webpgnm

echo "</font></td></tr></table>" >> $webpgpath/$webpgnm
##'---------------------'
## END MY_SYN_DROP table
##'---------------------'
fi ##END MY_SYN_DROP table delimiter bypass


##'------------------------------------'
##'------------------------------------'
## END horzontal YAMN Statistics tables
##'------------------------------------'
##'------------------------------------'
echo "</font></td></tr></table><br>" >> $webpgpath/$webpgnm


##'--------------------------'
##'--------------------------'
## BEGIN Pool horzontal table
##'--------------------------'
##'--------------------------'
echo "<table><tr valign=\"top\"><td>" >> $webpgpath/$webpgnm  # BEGIN


##'---------------------'
## BEGIN list Pool table
##'---------------------'
   echo "<table border=\"1\" cellpadding=\"2\" cellspacing=\"0\"><tr><td bgcolor=\"$titlecolor\">
   <font face=\"Verdana\" color=\"$fontcl\" size=\"$fontsz\"><b>Pool "-" $(find /home/yamn/yamn/pool -type f | wc -l) "-" $(date +"%r")</font></td></tr>
   <tr><td><font face=\"Courier New\" size=\"$fontsz\" color=\"$fontcolor\"><b>" >> $webpgpath/$webpgnm

## put count of each rec type here
   cat /dev/null > $filePath/templ.$fileoffset.txt
   for i in $yamnpath/pool/* ; do data=$(stat $i --format="%y") && echo "${data%.*} $i $(sed -n -e 2p $i)" >> $filePath/templ.$fileoffset.txt ; done

cat $filePath/templ.$fileoffset.txt

   sed -i 's/\/home\/yamn\/yamn\/pool\///g' $filePath/templ.$fileoffset.txt
   sed -e 's/$/<br>/' $filePath/templ.$fileoffset.txt | sort >> $webpgpath/$webpgnm  # append <br> to each line in templ.$fileoffset.txt
   echo "</font></td></tr></table><br>" >> $webpgpath/$webpgnm
##'---------------------'
## END list Pool table
##'---------------------'


echo "</td><td>" >> $webpgpath/$webpgnm  # MIDDLE


# display multiple web page hits begin
cat /dev/null > $filePath/temp20.$fileoffset.txt
cat /dev/null > $filePath/temp21.$fileoffset.txt
cat /dev/null > $filePath/AccessLogHits.$fileoffset.txt

cat /var/log/nginx/access.log > $filePath/temp19.$fileoffset.txt
cat /var/log/nginx/access.log.1 >> $filePath/temp19.$fileoffset.txt
tail -1000000 $filePath/temp19.$fileoffset.txt | awk '{print $1}' | sort | uniq -c |sort -n > $filePath/temp20.$fileoffset.txt

while read line1; do
      if [[ $(awk '{print $1}') -gt 40 ]] <<< $line1; then
         if [[ ! $line1 =~ "95.85.40.163" ]] && [[ ! $line1 =~ "127.0.0.1" ]] && [[ ! $line1 =~ "23.237.26.103" ]] &&  \
[[ ! $line1 =~ "50.26.242.136" ]] && [[ ! $line1 =~ "185.158.249.133" ]] && [[ ! $line1 =~ "194.76.225.113" ]]; then
            printf '%-5s %-17s\n' $(awk '{print $1" "$2}' <<< $line1) >> $filePath/temp21.$fileoffset.txt
         fi
      fi
done< $filePath/temp20.$fileoffset.txt # read file line by line

while read line1; do
      varip=$(awk "NR==1{print;exit}" <<< $line1 | whois $(awk '{print $2}') | grep country | awk '{print $2}' | sed '$!N; /^\(.*\)\n\1$/!P; D')  # pull out country code
      line2=$line1" "$varip
      varx1=$(printf '%-5s %-16s %-3s\n' $(awk '{print $1" "$2" "$3}' <<< $line2))
      varx2=${varx1// /\&nbsp\;}
      echo $varx2 >> $filePath/AccessLogHits.$fileoffset.txt
done< $filePath/temp21.$fileoffset.txt

if [ -e $filePath/AccessLogHits.$fileoffset.txt ] && [[ $(cat $filePath/AccessLogHits.$fileoffset.txt | wc -l) -gt 0 ]]; then
   echo "<table border=\"1\" cellpadding=\"2\" cellspacing=\"0\"><tr><td bgcolor=\"$titlecolor\">
   <font face=\"Verdana\" color=\"$fontcl\" size=\"$fontsz\"><b>Web access.log hits &gt 20</b></font></td></tr>
   <tr><td><font face=\"Courier New\" size=\"$fontsz\" color=\"$fontcolor\"><b>" >> $webpgpath/$webpgnm

   sed -e 's/$/<br>/' $filePath/AccessLogHits.$fileoffset.txt >> $webpgpath/$webpgnm
   echo "</font></td></tr></table><br>" >> $webpgpath/$webpgnm

   rm $filePath/temp19.$fileoffset.txt
   rm $filePath/temp20.$fileoffset.txt
   rm $filePath/temp21.$fileoffset.txt
fi
# display multiple web page hits end

echo "</td></tr></table>" >> $webpgpath/$webpgnm  # END
# //      <<<--------me only--------me only--------me only

# display mailq begin
vattest=$(mailq)
if [[ ! $vattest = "Mail queue is empty" ]]; then
   echo "<table border=\"1\" cellpadding=\"2\" cellspacing=\"0\"><tr><td bgcolor=\"$titlecolor\">
   <font face=\"Verdana\" color=\"$fontcl\" size=\"$fontsz\"><b>Mailq</b></font></td></tr>
   <tr><td><font face=\"Courier New\" size=\"$fontsz\" color=\"$fontcolor\"><b>" >> $webpgpath/$webpgnm

echo "$vattest" | sort | uniq -c | sort -nk1 | awk '{$1=$1}1' | sed '/^.\{10,50\}$/!d' | grep -v "Request" > $filePath/templ.$fileoffset.txt
sed -e 's/$/<br>/' $filePath/templ.$fileoffset.txt >> $webpgpath/$webpgnm
echo "<br>" >> $webpgpath/$webpgnm

echo "$vattest" > $filePath/templ.$fileoffset.txt
sed -e 's/$/<br>/' $filePath/templ.$fileoffset.txt >> $webpgpath/$webpgnm
echo "</font></td></tr></table><br>" >> $webpgpath/$webpgnm
fi
# display mailq end

echo "</body></html>" >> $webpgpath/$webpgnm

rm $filePath/temp.$fileoffset.txt
rm $filePath/temp1.$fileoffset.txt
rm $filePath/temp2.$fileoffset.txt
rm $filePath/templ.$fileoffset.txt
rm $filePath/templ2.$fileoffset.txt
rm $filePath/temp12.$fileoffset.txt
rm $filePath/temp13.$fileoffset.txt
rm $filePath/temp14.$fileoffset.txt
rm $filePath/temp16.$fileoffset.txt
rm $filePath/temp19.$fileoffset.txt
rm $filePath/temp20.$fileoffset.txt
rm $filePath/temp21.$fileoffset.txt

exit 0

# Ystats.sh
```
