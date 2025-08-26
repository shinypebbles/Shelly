# Shelly
Collect and process data from Shelly in an Oracle database

![Shelly Plus Plug S](https://github.com/shinypebbles/Shelly/blob/main/shellyplusplugs.png)

This project collects data from the [Shelly Plus Plug S](https://www.shelly.cloud/en/products/shop/shelly-plus-plug-s) and stores it in an Oracle database (v12.1). The data is displayed in a graph using Oracle APEX (v20.1).

The Shelly Plug delivers data in JSON, so let's create a data model to store the data. The owner of the tables is: SHELLY.

```
CREATE TABLE  "ELECTRICAL_APPLIANCE" 
   (	"EAP_ID" NUMBER NOT NULL ENABLE, 
	"NAME" VARCHAR2(40) NOT NULL ENABLE, 
	 CONSTRAINT "EAP_PK" PRIMARY KEY ("EAP_ID")
  USING INDEX  ENABLE
   )
/

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

CREATE TABLE  "CONNECTION" 
   (	"EAP_ID" NUMBER NOT NULL ENABLE, 
	"PLUGSEQ#" NUMBER NOT NULL ENABLE, 
	"STARTDATE" DATE NOT NULL ENABLE, 
	"ENDDATE" DATE, 
	 CONSTRAINT "CONNECTION_PK" PRIMARY KEY ("EAP_ID", "PLUGSEQ#", "STARTDATE")
  USING INDEX  ENABLE
   )
/
ALTER TABLE  "CONNECTION" ADD CONSTRAINT "CONNECTION_FK1" FOREIGN KEY ("EAP_ID")
	  REFERENCES  "ELECTRICAL_APPLIANCE" ("EAP_ID") ENABLE
/
ALTER TABLE  "CONNECTION" ADD CONSTRAINT "CONNECTION_FK2" FOREIGN KEY ("PLUGSEQ#")
	  REFERENCES  "PLUG" ("SEQ#") ENABLE
/
```

The table "ELECTRICAL_APPLIANCE" contains an applicance id and the name of the appliance. The parent table "PLUG" contains an ID called seq# and the hostname of the Shelly plug. The child table "PLUGSTATUS" contains a reference to the parent ID and the collected data in JSON-format. As a plug can be connected to various applicances over time there is a table called "CONNECTION" that contains a reference to an applicance and a plug with a time period.

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
The following procedure fetches the data of connected plugs and inserts it in the database.
```
create or replace PROCEDURE getplugstatus AS
  req   UTL_HTTP.REQ;
  resp  UTL_HTTP.RESP;
  value VARCHAR2(1024);
BEGIN
  for c1_rec in (select P.seq# 
    , P.hostname
    from plug P
       , connection C
    where P.seq#=C.plugseq#
    and C.enddate is null
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
        DBMS_OUTPUT.PUT_LINE('other error at plug: '||c1_rec.seq#);
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
begin
  return FROM_TZ(TO_TIMESTAMP('1970-01-01 00:00:00', 'YYYY-MM-DD HH24:MI:SS') + NUMTODSINTERVAL(pl_epoch, 'SECOND'), 'UTC') AT TIME ZONE 'CET';
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

Procedure to display current firmware version and available updates
```
CREATE OR REPLACE PROCEDURE getplugfirmware AS
  req              UTL_HTTP.REQ;
  resp             UTL_HTTP.RESP;
  json_response    CLOB;
  fw_id_value      VARCHAR2(100);
  device_name      VARCHAR2(100);
  version_only     VARCHAR2(20);
  available_beta   VARCHAR2(50);
  available_stable VARCHAR2(50);
BEGIN
  FOR c1_rec IN (
    SELECT P.seq#, P.hostname, P.name
      FROM plug P
      JOIN connection C ON P.seq# = C.plugseq#
     WHERE C.enddate IS NULL
     ORDER BY 1
  )
  LOOP
    BEGIN
      ----------------------------------------------------------------------
      -- 1st API call: Sys.GetConfig
      ----------------------------------------------------------------------
      req := UTL_HTTP.BEGIN_REQUEST('http://' || c1_rec.hostname || '/rpc/Sys.GetConfig');
      UTL_HTTP.SET_HEADER(req, 'User-Agent', 'Mozilla/4.0');
      resp := UTL_HTTP.GET_RESPONSE(req);

      json_response := NULL;
      BEGIN
        UTL_HTTP.READ_TEXT(resp, json_response);
      EXCEPTION
        WHEN UTL_HTTP.END_OF_BODY THEN
          NULL; -- complete response is in json_response
      END;
      UTL_HTTP.END_RESPONSE(resp);

      -- Parse fw_id and device name
      SELECT json_value(json_response, '$.device.fw_id'),
             json_value(json_response, '$.device.name')
        INTO fw_id_value, device_name
        FROM dual;

      SELECT REGEXP_SUBSTR(fw_id_value, '\d+\.\d+\.\d+')
        INTO version_only
        FROM dual;

      DBMS_OUTPUT.PUT_LINE('Plug ' || c1_rec.seq# || ' (' || c1_rec.name || ' / ' || device_name || ')');
      DBMS_OUTPUT.PUT_LINE('  Current firmware: ' || version_only);

      ----------------------------------------------------------------------
      -- 2nd API call: Sys.GetStatus
      ----------------------------------------------------------------------
      req := UTL_HTTP.BEGIN_REQUEST('http://' || c1_rec.hostname || '/rpc/Sys.GetStatus');
      UTL_HTTP.SET_HEADER(req, 'User-Agent', 'Mozilla/4.0');
      resp := UTL_HTTP.GET_RESPONSE(req);

      json_response := NULL;
      BEGIN
        UTL_HTTP.READ_TEXT(resp, json_response);
      EXCEPTION
        WHEN UTL_HTTP.END_OF_BODY THEN
          NULL;
      END;
      UTL_HTTP.END_RESPONSE(resp);

      -- Parse beta and stable update versions
      SELECT json_value(json_response, '$.available_updates.beta.version'),
             json_value(json_response, '$.available_updates.stable.version')
        INTO available_beta, available_stable
        FROM dual;

      IF available_beta IS NOT NULL THEN
        DBMS_OUTPUT.PUT_LINE('  Available BETA update: ' || available_beta);
      END IF;

      IF available_stable IS NOT NULL THEN
        DBMS_OUTPUT.PUT_LINE('  Available STABLE update: ' || available_stable);
      END IF;

      DBMS_OUTPUT.PUT_LINE('--------------------------------------------------');

    EXCEPTION
      WHEN OTHERS THEN
        DBMS_OUTPUT.PUT_LINE('Error on plug ' || c1_rec.seq# || ': ' || SQLERRM);
    END;
  END LOOP;
END;
/

```
