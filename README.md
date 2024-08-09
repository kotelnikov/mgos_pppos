# PPPoS / cellular modem support with addition functionality of receiving incoming calls and SMS notifications (command mode)

Original documentation is located [here](https://github.com/mongoose-os-libs/pppos).
The difference between the original library and this updated library is given below.

If you found this repository useful for your Mongoose OS project, please give this repository a star ⭐!

## Additional settings in `mos.yaml` file

```
  "pppos": {
    # Command mode settings.
    "stay_cmd_mode": false,   # The GSM module stays at command mode and receives calls & sms
    "decline_calls": false,   # Hangup incoming calls after receiving"
    "delete_sms": false       # Delete incoming sms after receiving"
    
    # Power settings for pwr key (if present).
    "pwr_gpio": -1 ,          # Power pin; -1 = no pwr key present
    "pwr_act",  0             # Active level, 0 or 1
  }
```

## Additional library events for handling incoming calls and SMS

```c
enum mgos_pppos_event {
  /* Existing early events */
  MGOS_PPPOS_GSM_READY,          // ev_data: struct mgos_pppos_info_arg
  MGOS_PPPOS_CALL,               // ev_data: struct mgos_pppos_call_arg
  MGOS_PPPOS_SMS,                // ev_data: struct mgos_pppos_sms_arg
};
```

## Application example (C)

*Configuration:*

```yaml
  - ["pppos.enable", true]
  - ["pppos.connect_on_startup", false]
  - ["pppos.stay_cmd_mode", true]
  - ["pppos.decline_calls", true]
  - ["pppos.delete_sms", true]
  - ["pppos.uart_no", 1]
  - ["pppos.rx_gpio", 26]
  - ["pppos.tx_gpio", 27]
  - ["pppos.pwr_gpio", 23]
  - ["pppos.pwr_act", 1]
  - ["pppos.rst_gpio", 5]
  - ["pppos.rst_act", 0]
```

*Application code:*

```c
#include "mgos.h"
#include "mgos_pppos.h"

void gsm_event_handler(int ev, void *ev_data, void *userdata) {
  switch (ev) {
    case MGOS_PPPOS_GSM_READY: {
      struct mgos_pppos_info_arg *info_arg = (struct mgos_pppos_info_arg *) ev_data;
      LOG(LL_INFO, ("GSM is ready, RSSI: %d", info_arg->rssi));
      break;
    }
    case MGOS_PPPOS_CALL: {
      struct mgos_pppos_call_arg *call_arg = (struct mgos_pppos_call_arg *) ev_data;
      LOG(LL_INFO, ("Incoming call notification CID: %s", call_arg->cid));
      
      // Send an SMS back to caller
      char *send_sms_cmd;
      mg_asprintf(&send_sms_cmd, 0, "AT+CMGS=\"%s\"", call_arg->cid);
      const struct mgos_pppos_cmd cmds[] = {
        {.cmd = send_sms_cmd},
        {.cmd = "Hello from MCU\x1A"},
        {.cmd = NULL}
      };
      mgos_pppos_run_cmds(0, cmds);
      free(send_sms_cmd);
      break;
    }
    case MGOS_PPPOS_SMS: {
      struct mgos_pppos_sms_arg *sms_arg = (struct mgos_pppos_sms_arg *) ev_data;
      LOG(LL_INFO, ("Incoming sms notification ID: %s, CID: %s, TIMESTAMP: %s, TEXT: %s", 
        sms_arg->id, sms_arg->cid, sms_arg->timestamp, sms_arg->data));
      break;
    }
    default: {
      break;
    }
  }
}


enum mgos_app_init_result mgos_app_init(void) {
  mgos_event_add_group_handler(MGOS_PPPOS_BASE, gsm_event_handler, NULL);
  return MGOS_APP_INIT_SUCCESS;
};

```