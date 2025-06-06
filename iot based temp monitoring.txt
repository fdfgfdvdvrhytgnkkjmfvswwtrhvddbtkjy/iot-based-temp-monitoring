
#include <stdio.h>
#include <string.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "esp_wifi.h"
#include "esp_event.h"
#include "esp_log.h"
#include "nvs_flash.h"
#include "esp_system.h"
#include "esp_netif.h"
#include "driver/adc.h"
#include "esp_http_client.h"

#define WIFI_SSID      "YourWiFiSSID"
#define WIFI_PASS      "YourWiFiPassword"
#define BLYNK_URL      "http://blynk.cloud/external/api/update?token=YourAuthToken&V0="

static const char *TAG = "TEMP_SENSOR";

// WiFi Event Handler
void wifi_init_sta(void) {
    esp_netif_init();
    esp_event_loop_create_default();
    esp_netif_create_default_wifi_sta();

    wifi_init_config_t wifi_init_cfg = WIFI_INIT_CONFIG_DEFAULT();
    esp_wifi_init(&wifi_init_cfg);

    wifi_config_t wifi_cfg = {
        .sta = {
            .ssid = WIFI_SSID,
            .password = WIFI_PASS,
        },
    };

    esp_wifi_set_mode(WIFI_MODE_STA);
    esp_wifi_set_config(ESP_IF_WIFI_STA, &wifi_cfg);
    esp_wifi_start();
    esp_wifi_connect();
}

// Send temperature data to Blynk cloud
void send_to_blynk(float temp) {
    char url[128];
    snprintf(url, sizeof(url), "%s%.2f", BLYNK_URL, temp);

    esp_http_client_config_t config = {
        .url = url,
    };
    esp_http_client_handle_t client = esp_http_client_init(&config);
    esp_http_client_perform(client);
    esp_http_client_cleanup(client);
}

void app_main() {
    nvs_flash_init();
    wifi_init_sta();

    adc1_config_width(ADC_WIDTH_BIT_12);
    adc1_config_channel_atten(ADC1_CHANNEL_6, ADC_ATTEN_DB_11); // GPIO34 = ADC1_CHANNEL_6

    while (1) {
        int raw = adc1_get_raw(ADC1_CHANNEL_6);
        float voltage = raw * 3.3 / 4095.0;
        float temperature = voltage * 100.0;

        ESP_LOGI(TAG, "Temperature: %.2f C", temperature);

        send_to_blynk(temperature);

        vTaskDelay(pdMS_TO_TICKS(1000)); // Delay 1 second
    }
}
