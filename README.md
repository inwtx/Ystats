# Ystats - YAMN Remailer Dashboard

This script can be used by a remailer sysops to monitor their YAMN server and remailer performance.  
Copy the script code below into a file called Ystats.sh and make it executable (sudo chmod 755 Ystats.sh).  
Then execute it with a cron as explained in at the top of the script code.  
The output can be accessed by: yourDN/Ystats.html  

```
#!/bin/bash
#
# Script to build server statistics Lstats.html
#
#------------------------------------------------------------------------#
#              --- Server statistics web page builder ---                #
#                                                                        #
# The following script can be used to monitor a YAMN remailer servers.   #
# Several statistics concerning the server and YAMN are displayed.       #
#                                                                        #
# Retrieve stastics for server web page - cron job                       #
# */1 * * * * /path/to/Ystats.sh                                         #
#                                                                        #
#------------------------------------------------------------------------#

#top
export PATH=$PATH:/sbin
export PATH=$PATH:/bin

webpgnm="/Ystats.html"
yamnpath="/home/yamn/yamn"       # no trailing /
webpgpath="/var/www/html"      # no trailing /
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
fontcolor=#00000040
titlecolor=#f0f0f0
machinewidth=430
roguewidth=200
freewidth=230
netstatswidth=230
mixmasterwidth=200
iptableswidth=125
miscstats=200
mixfiles=160
remailerstatistics=230
titlecolor=#f0f0f0
MachineRogueTableWidth=1112
MixMiscRemailerTableWidth=870


cat /dev/null > $webpgpath/$webpgnm  # clear html file

echo "<html><head><title>Server Stats</title></head><body bgcolor=\"$bgclr\" TEXT=\"$fontcolor\" LANG=\"en-US\" DIR=\"LTR\">" > $webpgpath/$webpgnm

##'-------------------'
## Top line date begin
##'-------------------'
echo "<font face=\"Verdana\" size=$fontsz color=\"$fontcolor\"><b>&nbsp;" >> $webpgpath/$webpgnm
echo $serverid $(date | cut -c 1-10 && date | cut -c 25-28 && echo "-" && date | cut -c 12-23) - Ystats >> $webpgpath/$webpgnm
echo "</font></b><br>" >> $webpgpath/$webpgnm
##'-----------------'
## Top line date end
##'-----------------'


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
<b><font face=\"Verdana\" size=$fontsz><b>Machine</b></font></td></tr>
<tr><td><font face=\"Courier New\" size=$fontsz color=\"$fontcolor\"><b>" >> $webpgpath/$webpgnm

## linux info
echo "$(lsb_release -d) $(uname -m)<br>" | sed -e 's/Description://g' | tr -d "\t" >> $webpgpath/$webpgnm

# ip address
echo "IP server: $(hostname -I)<br>" >> $webpgpath/$webpgnm

# domain name
echo "Domain name: $(dig -x $(hostname -I) +short | awk -F '.' '{print $1"."$2}')<br>" >> $webpgpath/$webpgnm

# MTU
echo "$(ifconfig | sed '1!d' | awk '{print "MTU: " $4}')<br>" >> $webpgpath/$webpgnm

## openssl
echo "$(openssl version)" | awk '{print $1" "$2" ("$3" "$4" "$5")<br>"}' >> $webpgpath/$webpgnm

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
   var2=$(date +"'%g")
   var3=$var1$var2
   var4=$(grep -e "$var3" <<< $(vnstat -m))
   bline=$(awk '{print "MTD bandwidth: " $1" "$2" - "$9" "$10}' <<<$var4)
   echo "<br>$bline" >> $webpgpath/$webpgnm
else
   echo "<br>MTD bandwidth: vnstat not running!" >> $webpgpath/$webpgnm
fi

## last login
(last -a root) > $filePath/templ.txt
sed -n -e 1p $filePath/templ.txt > $filePath/templ2.txt
(cat $filePath/templ2.txt | colrm 1 22 | colrm 35 100) > $filePath/templ.txt
lines=$(head -n 1 $filePath/templ.txt)
echo "<br>Last login: $lines" >> $webpgpath/$webpgnm
echo "</font></b></td></tr></table><br>" >> $webpgpath/$webpgnm    # end build Machine
##'-----------------'
## END Machine table
##'-----------------'


##'------------------------------------'
echo "</td><td>" >> $webpgpath/$webpgnm  # MIDDLE: horzontal divider for Machine and Netstat/Free
##'------------------------------------'


##'-------------------'
## BEGIN netstat table
##'-------------------'
echo "<table border=\"1\" cellpadding=\"2\" cellspacing=\"0\"><tr><td bgcolor=\"$titlecolor\">
<font face=\"Verdana\" size=$fontsz><b>Netstats</b></font></td></tr>
<tr><td><font face=\"Courier New\" size=$fontsz color=\"$fontcolor\"><b>" >> $webpgpath/$webpgnm
netstat -vatnp > $filePath/templ.txt
sed -i "/tcp6/d" $filePath/templ.txt            # remove tcp6 lines
sed -i "/python2.7/d" $filePath/templ.txt       # remove bitmessage python2.7 messages
sed -i "/nginx\: worker/d" $filePath/templ.txt  # remove nginx: worker messages
sed -i "/TIME_WAIT/d" $filePath/templ.txt       # remove TIME_WAIT messages
sed -i 's/ /\&nbsp;/g' $filePath/templ.txt
sed -i 's/ //g' $filePath/templ.txt
sed -e 's/$/<br>/' $filePath/templ.txt >> $webpgpath/$webpgnm
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
<font face=\"Verdana\" size=$fontsz><b>Free</b></font></td></tr>
<tr><td><font face=\"Courier New\" size=$fontsz color=\"$fontcolor\"><b>" >> $webpgpath/$webpgnm

free > $filePath/templ.txt
sed -i 's/ /\&nbsp;/g' $filePath/templ.txt
sed -i 's/ //g' $filePath/templ.txt
sed -e 's/$/\&nbsp;\&nbsp;<br>/' $filePath/templ.txt >> $webpgpath/$webpgnm  # append <br>
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
<font face=\"Verdana\" size=$fontsz><b>Misc Stats</b></font></td></tr>
<tr><td><font face=\"Courier New\" size=$fontsz color=\"$fontcolor\"><b>" >> $webpgpath/$webpgnm

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
if [ -e $filePath/AccessLogHits.txt ] && [[ $(cat $filePath/AccessLogHits.txt | wc -l) -gt 0 ]]; then
   tempvar3=$(wc -l $filePath/AccessLogHits.txt | awk '{print $1}')
   tempvar=`echo "<font color=FF0000><b>Hits:&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;$tempvar3</b></font>" && echo "<br>"`
   echo $tempvar >> $webpgpath/$webpgnm
fi
echo "</font></td></tr></table><br>" >> $webpgpath/$webpgnm   # end build Misc
##'--------------------'
## END Misc Stats table
##'--------------------'


##'------------------------------------'
echo "</td><td>" >> $webpgpath/$webpgnm  # MIDDLE: horzontal divider for Misc Stats & YAMN Stats
##'------------------------------------'


##'----------------------'
## BEGIN YAMN Stats table
##'----------------------'
#echo "<table border=\"1\" cellpadding=\"2\" cellspacing=\"0\"><tr><td width=\"$mixmasterwidth\" bgcolor=\"$titlecolor\">
echo "<table border=\"1\" cellpadding=\"2\" cellspacing=\"0\"><tr><td bgcolor=\"$titlecolor\">
<font face=\"Verdana\" size=$fontsz><b>YAMN Stats</b></font></td></tr>
<tr><td><font face=\"Courier New\" size=$fontsz color=\"$fontcolor\"><b>" >> $webpgpath/$webpgnm

## mailq
echo "mailq count: " > $filePath/templ.txt
vattest=$(mailq)
if [[ $vattest = "Mail queue is empty" ]]; then
   cat /dev/null > $filePath/notification.txt
   echo "0" >> $filePath/templ.txt
   varzero="y"
   else
   varzero="n"
   mailq | grep -c "^[A-Z0-9]" >> $filePath/templ.txt
   varmq=$(mailq | grep -c "^[A-Z0-9]")
fi

sed '1~2 {N;N;s/\n/ /g}' $filePath/templ.txt > $filePath/templ2.txt  # concat lines 1 and 2
if [[ $varzero = "n" ]]; then
   vartemp=$(<$filePath/templ2.txt)
   echo "<font color=FF0000><b>"$vartemp"</b></font>" > $filePath/templ2.txt  # color 'mailq count:' red
fi

cat $filePath/templ2.txt >> $webpgpath/$webpgnm
echo "<br>" >> $webpgpath/$webpgnm
## mailq end

## pool count
$filePath/poolcount.sh

echo "pool count:&nbsp;" > $filePath/templ.txt
find $yamnpath/pool -type f | wc -l >> $filePath/templ.txt
sed '1~2 {N;N;s/\n/ /g}' $filePath/templ.txt > $filePath/templ2.txt  # concat lines 1 and 2
cat $filePath/templ2.txt >> $webpgpath/$webpgnm
echo "<br>" >> $webpgpath/$webpgnm
## pool count end

## pool today and yesterday
if [ ! -s $filePath/savetodaypoolcnt.txt ]; then
   ptd=0
   pyd=0
   echo "0" > $filePath/savetodaypoolcnt.txt
   echo "0" >> $filePath/savetodaypoolcnt.txt
fi

if [ -e $filePath/poolcount.sh ] && [[ $(cat $filePath/savetodaypoolcnt.txt | wc -l) -gt 0 ]]; then  # running poolcount.sh?
   ptd=$(head -n 1 $filePath/savetodaypoolcnt.txt)  # get 1st line = total thus far today
   pyd=$(sed -n 2p $filePath/savetodaypoolcnt.txt)  # get 2nd line = prior
   echo "pool today: &nbsp;$ptd<br>" >> $webpgpath/$webpgnm
   echo "pool prior: &nbsp;$pyd<br>" >> $webpgpath/$webpgnm
fi
## pool today and yesterday end

echo "</font></td></tr></table><br>" >> $webpgpath/$webpgnm   # end build Misc
##'--------------------'
## END YAMN Stats table
##'--------------------'


##'------------------------------------'
echo "</td><td>" >> $webpgpath/$webpgnm  # MIDDLE: horzontal divider for YAMN Stats & YAMN
##'------------------------------------'


##'----------------'
## BEGIN YAMN table
##'----------------'
#echo "<table border=\"1\" cellpadding=\"2\" cellspacing=\"0\"><tr><td width=\"$mixmasterwidth\" bgcolor=\"$titlecolor\">
echo "<table border=\"1\" cellpadding=\"2\" cellspacing=\"0\"><tr><td bgcolor=\"$titlecolor\">
<font face=\"Verdana\" size=$fontsz><b>YAMN</b></font></td></tr>
<tr><td><font face=\"Courier New\" size=$fontsz color=\"$fontcolor\"><b>" >> $webpgpath/$webpgnm

## top stats
top -b -n1 > $filePath/templ.txt             # get TOP listing of exes
var1=$(grep 'PID USER' $filePath/templ.txt)  # get header line
var2=$(grep -w yamn $filePath/templ.txt)        # get yamn line
echo "$var1" > $filePath/templ.txt
echo "$var2" >> $filePath/templ.txt
sed -i 's/ /\&nbsp;/g' $filePath/templ.txt
sed -e 's/$/<br>/' $filePath/templ.txt >> $webpgpath/$webpgnm  # append <br>
echo "<br>" >> $webpgpath/$webpgnm

## start date/time
#echo "$(grep "yamn" <<< "$(ps -eo lstart,cmd)")" | awk '{print "Started: "$1" "$2" "$3" "$4}' >> $webpgpath/$webpgnm
echo $(grep "yamn" <<< "$(ps -eo lstart,cmd)") | awk '{print "Started: "$1" "$2" "$3" "$4}' | cut -c 1-28 >> $webpgpath/$webpgnm

## md5
echo "<br>MD5: $(md5sum $yamnpath/yamn)" > $filePath/templ.txt
cat $filePath/templ.txt >> $webpgpath/$webpgnm

## key(s)
#var1="<br>Seckey: "
var1="<br>Pub/Sec key: "
grep "Created:" $yamnpath/secring.mix > $filePath/temp1.txt  # extract Created: date
grep "Expires:" $yamnpath/secring.mix > $filePath/temp12.txt  # extract Expires: date

cat $filePath/temp12.txt > $filePath/temp16.txt
sed -i 's/Expires: //g' $filePath/temp16.txt  # del Expires:

paste $filePath/temp1.txt $filePath/temp12.txt > $filePath/temp13.txt  # stack dates side by side
sed -i 's/\t/ /g' $filePath/temp13.txt              # replace tab btwn dates with space
sed -i 's/Created: //g' $filePath/temp13.txt
sed -i 's/Expires: //g' $filePath/temp13.txt
var7=$(awk 'c&&!--c;/Expires:/{c=1'} $yamnpath/secring.mix)
echo "$var7" > $filePath/temp14.txt
var2=0

while read line1; do
   ((var2++))
   var3=$(awk -F '[ \t\n\v\r.]' '{print $1" "$2}' <<< $line1)  # get 2014-05-08 2014-11-04
   var9=$(awk -v var8=$var2 'NR==var8' $filePath/temp16.txt)  # pull stacked dates from temp12.txt
   var10=$(date +"%G-%m-%d")
   days=$(( ($(date --date=$var9 +%s) - $(date --date=$var10 +%s) )/(60*60*24) ))   # calc days between
   var5=$days
   var7=$(sed -n $var2'p' $filePath/temp14.txt)
   var4="$var1 $var3 ($var5) ($var7)"                    #<- days remaining calc
   var4="$var1 $var3 ($var7)"                    #<- days remaining calc
   echo $var4 >> $webpgpath/$webpgnm
done<$filePath/temp13.txt

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


##'------------------------------'
## BEGIN YAMN stat source I table
##'------------------------------'
echo "<table border=\"1\" cellpadding=\"2\" cellspacing=\"0\"><tr><td bgcolor=\"$titlecolor\">
<font face=\"Verdana\" size=$fontsz><b>YAMN Statistics (mixmin)</b></font></td></tr>
<tr><td><font face=\"Courier New\" size=$fontsz color=\"$fontcolor\"><b>" >> $webpgpath/$webpgnm

if [[ $(date +"%M") = "00" ]] || \
   [[ $(date +"%M") = "10" ]] || \
   [[ $(date +"%M") = "20" ]] || \
   [[ $(date +"%M") = "30" ]] || \
   [[ $(date +"%M") = "40" ]] || \
   [[ $(date +"%M") = "50" ]] || \
   [[ ! -s $filePath/astats.txt ]] || [[ $dostats = "y" ]]; then
   sleep 5  # pause on 1st stat collect for pingers to finish updating their stats
   wget  --no-check-certificate --timeout=15  -t 1 http://www.mixmin.net/yamn/mlist2.txt -O $filePath/mixmin.txt
   echo $(date) > $filePath/statdate.txt
fi

savdate=$(< $filePath/statdate.txt)
grep "%"  $filePath/mixmin.txt | colrm 14 27 > $filePath/astats.txt

sed -i 's/\?/ /g' $filePath/astats.txt
sed -i 's/ *$//' $filePath/astats.txt
sed -i 's/ /\&nbsp;/g' $filePath/astats.txt
sed -i 's/^/\&nbsp;/' $filePath/astats.txt       # prepend a blank
sed -i 's/$/\&nbsp;/' $filePath/astats.txt       # append a blank
sed -i "1i&nbsp;$savdate" $filePath/astats.txt
sed -i 's/$/<br>/' $filePath/astats.txt

rm $filePath/templ.txt
cat $filePath/astats.txt >> $webpgpath/$webpgnm
echo "</font></td></tr></table>" >> $webpgpath/$webpgnm
##'----------------------------'
## END YAMN stat source I table
##'----------------------------'


##'------------------------------------'
echo "</td><td>" >> $webpgpath/$webpgnm  # MIDDLE: horzontal divider for source I & II table
##'------------------------------------'


##'-------------------------------'
## BEGIN YAMN stat source II table
##'-------------------------------'
if true; then
echo "<table border=\"1\" cellpadding=\"2\" cellspacing=\"0\"><tr><td bgcolor=\"$titlecolor\">
<font face=\"Verdana\" size=$fontsz><b>YAMN Statistics (sec3)</b></font></td></tr>
<tr><td><font face=\"Courier New\" size=$fontsz color=\"$fontcolor\"><b>" >> $webpgpath/$webpgnm

if [[ $(date +"%M") = "00" ]] || \
   [[ $(date +"%M") = "10" ]] || \
   [[ $(date +"%M") = "20" ]] || \
   [[ $(date +"%M") = "30" ]] || \
   [[ $(date +"%M") = "40" ]] || \
   [[ $(date +"%M") = "50" ]] || \
   [[ ! -s $filePath/astats.txt ]] || [[ $dostats = "y" ]]; then
   wget  --no-check-certificate --timeout=15  -t 1 https://cloaked.pw/yamn/mlist2.txt -O $filePath/sec3.txt
   echo $(date) > $filePath/statdate.txt
fi

savdate=$(< $filePath/statdate.txt)
grep "%"  $filePath/sec3.txt | colrm 14 27 > $filePath/astats.txt
sed -i 's/\?/ /g' $filePath/astats.txt
sed -i 's/ *$//' $filePath/astats.txt
sed -i "s/?/ /15" $filePath/astats.txt
sed -i 's/ /\&nbsp;/g' $filePath/astats.txt
sed -i 's/^/\&nbsp;/' $filePath/astats.txt       # prepend a blank
sed -i 's/$/\&nbsp;/' $filePath/astats.txt       # append a blank
sed -i "1i&nbsp;$savdate" $filePath/astats.txt
sed -i 's/$/<br>/' $filePath/astats.txt

rm $filePath/templ.txt
cat $filePath/astats.txt >> $webpgpath/$webpgnm
echo "</font></td></tr></table>" >> $webpgpath/$webpgnm
##'-----------------------------'
## END YAMN stat source II table
##'-----------------------------'


##'------------------------------------'
echo "</td><td>" >> $webpgpath/$webpgnm  # MIDDLE: horzontal divider for source II & III table
##'------------------------------------'


##'--------------------------------'
## BEGIN YAMN stat source III table
##'--------------------------------'
echo "<table border=\"1\" cellpadding=\"2\" cellspacing=\"0\"><tr><td bgcolor=\"$titlecolor\">
<font face=\"Verdana\" size=$fontsz><b>YAMN Statistics (talc)</b></font></td></tr>
<tr><td><font face=\"Courier New\" size=$fontsz color=\"$fontcolor\"><b>" >> $webpgpath/$webpgnm

if [[ $(date +"%M") = "00" ]] || \
   [[ $(date +"%M") = "10" ]] || \
   [[ $(date +"%M") = "20" ]] || \
   [[ $(date +"%M") = "30" ]] || \
   [[ $(date +"%M") = "40" ]] || \
   [[ $(date +"%M") = "50" ]] || \
   [[ ! -s $filePath/astats.txt ]] || [[ $dostats = "y" ]]; then
   wget  --no-check-certificate --timeout=15  -t 1 https://talcserver.com/yamn/mlist2.txt -O $filePath/talc.txt
   echo $(date) > $filePath/statdate.txt
fi

savdate=$(< $filePath/statdate.txt)
grep "%"  $filePath/talc.txt | colrm 14 27 > $filePath/astats.txt
sed -i 's/\?/ /g' $filePath/astats.txt
sed -i 's/ *$//' $filePath/astats.txt
sed -i 's/ /\&nbsp;/g' $filePath/astats.txt
sed -i 's/^/\&nbsp;/' $filePath/astats.txt       # prepend a blank
sed -i 's/$/\&nbsp;/' $filePath/astats.txt       # append a blank
sed -i "1i&nbsp;$savdate" $filePath/astats.txt
sed -i 's/$/<br>/' $filePath/astats.txt

rm $filePath/templ.txt

cat $filePath/astats.txt >> $webpgpath/$webpgnm
echo "</font></td></tr></table>" >> $webpgpath/$webpgnm
##'------------------------------'
## END YAMN stat source III table
##'------------------------------'


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
   <font face=\"Verdana\" size=$fontsz><b>Pool "-" $(find $yamnpath/pool -type f | wc -l) "-" $(date +"%r")</font></td></tr>
   <tr><td><font face=\"Courier New\" size=$fontsz color=\"$fontcolor\"><b>" >> $webpgpath/$webpgnm

## put count of each rec type here
   cat /dev/null > $filePath/templ.txt
   for i in $yamnpath/pool/* ; do data=$(stat $i --format="%y") && echo "${data%.*} $i $(sed -n -e 2p $i)" >> $filePath/templ.txt ; done

cat $filePath/templ.txt

   sed -i 's/\/home\/yamn\/yamn\/pool\///g' $filePath/templ.txt
   sed -e 's/$/<br>/' $filePath/templ.txt | sort >> $webpgpath/$webpgnm  # append <br> to each line in templ.txt
   echo "</font></td></tr></table><br>" >> $webpgpath/$webpgnm
##'---------------------'
## END list Pool table
##'---------------------'

echo "</td></tr></table>" >> $webpgpath/$webpgnm  # END

# display mailq begin
vattest=$(mailq)
if [[ ! $vattest = "Mail queue is empty" ]]; then
   echo "<table border=\"1\" cellpadding=\"2\" cellspacing=\"0\"><tr><td bgcolor=\"$titlecolor\">
   <font face=\"Verdana\" size=$fontsz><b>Mailq</b></font></td></tr>
   <tr><td><fo```nt face=\"Courier New\" size=$fontsz color=\"$fontcolor\"><b>" >> $webpgpath/$webpgnm

echo "$vattest" | sort | uniq -c | sort -nk1 | awk '{$1=$1}1' | sed '/^.\{10,50\}$/!d' | grep -v "Request" > $filePath/templ.txt
sed -e 's/$/<br>/' $filePath/templ.txt >> $webpgpath/$webpgnm
echo "<br>" >> $webpgpath/$webpgnm

echo "$vattest" > $filePath/templ.txt
sed -e 's/$/<br>/' $filePath/templ.txt >> $webpgpath/$webpgnm
echo "</font></td></tr></table><br>" >> $webpgpath/$webpgnm
fi
# display mailq end

echo "</body></html>" >> $webpgpath/$webpgnm

rm $filePath/temp.txt
rm $filePath/temp1.txt
rm $filePath/temp2.txt
rm $filePath/templ.txt
rm $filePath/templ2.txt
rm $filePath/temp12.txt
rm $filePath/temp13.txt
rm $filePath/temp14.txt
rm $filePath/temp16.txt
rm $filePath/temp19.txt
rm $filePath/temp20.txt
rm $filePath/temp21.txt

exit 0

# Ystats.sh
