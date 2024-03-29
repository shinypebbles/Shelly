# Shelly
Collect and process data from Shelly in an Oracle database

![Shelly Plus Plug S](https://github.com/shinypebbles/Shelly/blob/main/shellyplusplugs.png)

This project collects data from the [Shelly Plus Plug S](https://www.shelly.cloud/en/products/shop/shelly-plus-plug-s) and stores it in an Oracle database (v12.1). The data is displayed in a graph using Oracle APEX (v20.1).

The Shelly Plug delivers data in JSON, so let's create a data model to store the data. The owner of the tables is: SHELLY.

```
CREATE TABLE  "PLUG" 
   (	"SEQ#" NUMBER NOT NULL ENABLE, 
	"HOSTNAME" VARCHAR2(40) NOT NULL ENABLE, 
	"NAME" VARCHAR2(40), 
	 CONSTRAINT "PLUG_PK" PRIMARY KEY ("SEQ#")
  USING INDEX  ENABLE
   )
/

CREATE TABLE  "PLUGSTATUS" 
   (	"PLUGSEQ#" NUMBER, 
	"DATA" CLOB, 
	 CONSTRAINT "PLUGSTATUS_JSON_CHK" CHECK (data IS JSON) ENABLE
   )
/
ALTER TABLE  "PLUGSTATUS" ADD CONSTRAINT "PLUGSTATUS_FK1" FOREIGN KEY ("PLUGSEQ#")
	  REFERENCES  "PLUG" ("SEQ#") ENABLE
/

CREATE INDEX  "PLUGSTATUS_IND1" ON  "PLUGSTATUS" ("PLUGSEQ#")
/
```

The parent table contains an ID called seq#, the hostname of the Shelly plugs and a name, for example 'freezer'. The child table contains a reference to the parent ID and the collected data in JSON-format.

The data is collected through a http REST API call. First we need to allow the table owner SHELLY http-access to hosts in the network.

```
BEGIN
  DBMS_NETWORK_ACL_ADMIN.append_host_ace (
    host       => '*', 
    lower_port => 80,
    upper_port => 80,
    ace        => xs$ace_type(privilege_list => xs$name_list('http'),
                              principal_name => 'SHELLY',
                              principal_type => xs_acl.ptype_db)); 
END;
/
```
Documentation of the [JSON-format](https://shelly-api-docs.shelly.cloud/gen2/ComponentsAndServices/Switch/#status). Here is an example of the returned JSON:
```
{"id":0, "source":"init", "output":true, "apower":0.0, "voltage":236.4, "current":0.047, "aenergy":{"total":3158.258,"by_minute":[0.000,0.000,0.000],"minute_ts":1684960207},"temperature":{"tC":37.5, "tF":99.5}}
```
The following procedure fetches the data and inserts it in the database.
```
create or replace PROCEDURE getplugstatus AS
  req   UTL_HTTP.REQ;
  resp  UTL_HTTP.RESP;
  value VARCHAR2(1024);
BEGIN
  for c1_rec in (select seq# 
    , hostname
    from plug
    order by 1)
  LOOP
    BEGIN
      req := UTL_HTTP.BEGIN_REQUEST('http://'||c1_rec.hostname||'/rpc/Switch.GetStatus?id=0');
      UTL_HTTP.SET_HEADER(req, 'User-Agent', 'Mozilla/4.0');
      resp := UTL_HTTP.GET_RESPONSE(req);
      LOOP
        UTL_HTTP.READ_LINE(resp, value, TRUE);
        DBMS_OUTPUT.PUT_LINE(value);
        insert into plugstatus(plugseq#, data) values (c1_rec.seq#, value);
      END LOOP;
      UTL_HTTP.END_RESPONSE(resp);
    EXCEPTION
      WHEN UTL_HTTP.END_OF_BODY THEN
        UTL_HTTP.END_RESPONSE(resp);
      WHEN others THEN
        DBMS_OUTPUT.PUT_LINE('other error');
    END;
  END LOOP;
END;
```
Create a job to collect the data every minute.
```
BEGIN
    DBMS_SCHEDULER.CREATE_JOB (
            job_name => '"SHELLY"."JOB_GET_STATUS"',
            job_type => 'PLSQL_BLOCK',
            job_action => 'begin getplugstatus; end;',
            number_of_arguments => 0,
            start_date => TO_TIMESTAMP_TZ('2015-11-17 21:35:00.000000000 EUROPE/AMSTERDAM','YYYY-MM-DD HH24:MI:SS.FF TZR'),
            repeat_interval => 'FREQ=MINUTELY',
            end_date => NULL,
            enabled => FALSE,
            auto_drop => TRUE,
            comments => 'Get Shelly status every minute');
 
    DBMS_SCHEDULER.SET_ATTRIBUTE( 
             name => '"SHELLY"."JOB_GET_STATUS"', 
             attribute => 'store_output', value => TRUE);
    DBMS_SCHEDULER.SET_ATTRIBUTE( 
             name => '"SHELLY"."JOB_GET_STATUS"', 
             attribute => 'logging_level', value => DBMS_SCHEDULER.LOGGING_OFF);
      
    DBMS_SCHEDULER.enable(
             name => '"SHELLY"."JOB_GET_STATUS"');
END;
```
The JSON minute_ts field is in Unix epoch format in UTC. So it needs to be converted to CET or CEST. For this a function is created.
```
create or replace function epoch2cet_cest
( pl_epoch in number)
  return date
is
  pl_date date;
  pl_start date;
  pl_end date;
  pl_offset number;
begin
  pl_date:=TO_DATE('1970-01-01', 'YYYY-MM-DD') + NUMTODSINTERVAL(pl_epoch, 'SECOND');
  pl_start:=NEXT_DAY(to_date('24-03-'||to_char(sysdate, 'YYYY'), 'DD-MM-YYYY'),'SUNDAY');
  pl_end:=NEXT_DAY(to_date('24-10-'||to_char(sysdate, 'YYYY'), 'DD-MM-YYYY'),'SUNDAY');
  if pl_date>pl_start and pl_date<pl_end then
    pl_offset:=2; --Central European Summer Time (CEST)
  else
    pl_offset:=1; --Central European Time (CET)
  end if;
  return pl_date+pl_offset/24;
end epoch2cet_cest;
```
The data needed for the graph is selected from the JSON data in the child table. 
```
select to_number(S.data.apower) as power
, epoch2cet_cest(S.data.aenergy.minute_ts) as date_time
, P.seq#
from plug P, plugstatus S
where P.seq#=S.plugseq#
  and P.seq#=:P88_SEQ
  and trunc(epoch2cet_cest(S.data.aenergy.minute_ts))=trunc(to_date(:P88_DATUM))
order by epoch2cet_cest(S.data.aenergy.minute_ts)
```

![Freezer power consumption](https://github.com/shinypebbles/Shelly/blob/main/Freezer.png)

Line graph of the freezer power consumption in one day.
