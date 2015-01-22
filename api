#!/usr/bin/env python
# -*- coding: utf-8 -*-


import cmd
import calendar
import datetime
import ast
import argparse
from pprint import pprint

import pyzabbix
import prettytable

import pyzapiconfig

file = open('results.html', 'w')
data = {}

parser = argparse.ArgumentParser(add_help=False)
parser.add_argument('-hi', '--hostid', action='append', dest='hostid', help='HostID')
parser.add_argument('-ii', '--itemid', action='append', dest='itemid', help='ItemID')
parser.add_argument('-hn', '--hostname', action='append', dest='hostname', help='HostName')
parser.add_argument('-in', '--itemname', action='append', dest='itemname', help='ItemName')
parser.add_argument('-th', '--hours', action='store', dest='time', type=int, help='Time')

def parse(line):
    args, unknown = parser.parse_known_args(line)
    if len(unknown) != 0:
        print "Неизвестные аргументы: '%s'" % str(unknown)
    return args

class Zapi(cmd.Cmd):

    def do_connect(self, line):
        global username, password, url
        if not username:
            username = raw_input('Имя пользователя: ')
        if not password:
            password = raw_input('Пароль: ')
        if not url:
            url = raw_input('Zabbix URL: ')
        global zapi
        zapi = pyzabbix.ZabbixAPI(url)
        zapi.login(username, password)
        print "Connected to Zabbix API Version %s" % zapi.api_version()

    def do_search_hosts(self, line):
        global zapi
        args=parse(line.split())
        hosts = zapi.host.get(
            output=['host', 'hostid'],
            search={'host': ','.join(args.hostname)})
        table = prettytable.PrettyTable(['Hostname', 'HostID'])
        table.align['Hostname'] = 'l'
        for item in hosts:
            table.add_row([item['host'], item['hostid']])
        print table

    def do_search_items(self, line):
        global zapi
        args=parse(line.split())
        items = zapi.item.get(output=['name', 'itemid', 'delay', 'hostid', 'key_', ], hostids=args.hostid, search={'name': ','.join(args.itemname)})

        hosts={}
        for item in items:
            hosts[str(item['hostid'])] = ''

        for item in hosts.keys():
            hosts[item] = zapi.host.get(output=['host', 'hostid'], hostids=item)[0]['host']

        table = prettytable.PrettyTable(['ItemName', 'ItemID', 'HostName','HostID'])
        table.align['ItemName'] = 'l'
        for item in items:
            table.add_row([item['name'], item['itemid'], hosts[item['hostid']], item['hostid']])
        print table

    def do_get_item(self, line):
        global zapi
        global data
        args=parse(line.split())
        for itemID in args.itemid:
            hists = zapi.history.get(output='extend', itemids=itemID, time_from=(calendar.timegm((datetime.datetime.utcnow()-datetime.timedelta(hours = args.time)).utctimetuple())))
            print '=' * 10 + 'ItemID: ' + itemID + '=' * 10
            table = prettytable.PrettyTable(['Time', 'Value'])
            table.align['Time'] = 'l'

            data[itemID]=[]

            for item in hists:
                data[itemID].append("{'time':'%s', 'value':'%s'}" % (item['clock'], item['value']))

            for item in data[itemID]:
                i = ast.literal_eval(item)
                table.add_row([i['time'], i['value']])

            print table

    def do_print_data(self, line):
        pprint (data)

    def do_rec(self, line):

        global data, file
        text = '''<html>
    <head>
        <script type="text/javascript" src="https://www.google.com/jsapi?autoload={'modules':[{'name':'visualization', 'version':'1', 'packages':['corechart']}]}"></script>

        <script type="text/javascript">
            google.setOnLoadCallback(drawChart);

            function drawChart(){
                var dateFormatter = new google.visualization.DateFormat({pattern: '%Y-%m-%d %H:%M:%S'});
                '''

        for result in data.keys():
            text += '''
                var data{0} = new google.visualization.DataTable(); 
                data{0}.addColumn('datetime', 'Time');
                data{0}.addColumn('number', '{0}');
                data{0}.addRows(['''.format(str(result))

            for item in data[result]:
                i = ast.literal_eval(item)
                text+="[new Date(%s), %s],\n" % (int(i['time']) * 1000, i['value'])

            text += '''        ]);'''

        text +='''
                var options = {
                    width: 1000,
                    height: 563,
                    pointSize: 4,
                    curveType: 'none',
                    interpolateNulls: true,
                    hAxis: { title: 'Time', format: 'yyyy.MM.dd H:mm', gridlines: {count: '-1' }},
                    vAxis: { title: 'Value', gridlines: {count: '-1' }}
                }

                var chart = new google.visualization.LineChart(document.getElementById('ex0'));

                google.visualization.events.addListener(chart, 'ready', function () {
                    png.innerHTML = '<img src="' + chart.getImageURI() + '">';
                });
                '''

        for i, result in enumerate(data.keys()):
            if i != 0:
                text += '''var Data = google.visualization.data.join(data, data%s, 'full', [[0, 0]], [1], [1]);
            ''' % (str(result))
            else:
                text += '''var data = data%s;
                ''' % data.keys()[0]

        text +='''
                chart.draw(Data, options);
            }
        </script>
    </head>
    <body>
        <div id="ex0"></div>
        <br /> <br />
        <div id="png"></div>
    </body>
</html>'''

        file.write(text)

    def do_EOF(self, line):
        return True

if __name__ == '__main__':
    Zapi().cmdloop()
