# Collect metrics with Prometheus with push gateway

## Run Push Gateway

    docker pull prom/pushgateway
    docker run --restart=unless-stopped -d -p 9091:9091 prom/pushgateway

## Setup scraping target in Prometheus

```yaml
job_name: TempBatch
static_configs:
  - targets: ["raspberrypi:9091"]
honor_labels: true
```

## Run as a service (passive mode)

One process per receiver reads every sensor of `mysensors.ini` (same format as `sensors.ini`) from their BLE advertisements, pvvx Custom or BTHome v2 format:

    git clone https://github.com/Poil/MiTemperature2 ~/pi/MiTemperature2 && cd ~/pi/MiTemperature2
    sudo apt install python3-venv python3-bluez libbluetooth-dev
    python3 -m venv --system-site-packages . && bin/pip install prometheus_client pycryptodomex requests bleak
    sudo cp prometheus/misensor.service /etc/systemd/system/ && sudo systemctl enable --now misensor

`--bthome-onlyfull-transmit` matters for BTHome sensors: temperature/humidity and battery come in separate packets, and without it the missing values are pushed as 0.

## Update the sensor firmware from the receiver

`ota.py` flashes the latest [pvvx firmware](https://github.com/pvvx/ATC_MiThermometer) over BLE, without phone or browser, and only to sensors older than the latest release. The image is chosen from the hardware id reported by the sensor. Stop the collector first, as both use the Bluetooth adapter:

    sudo systemctl stop misensor
    bin/python ota.py --check -df mysensors.ini   # model and installed version
    bin/python ota.py -df mysensors.ini           # about 4 minutes per sensor
    sudo systemctl start misensor

A transfer that is cut short is harmless: the sensor only switches to the new image after its CRC check.

## Grafana Dashboard

![grafana-dashboard](https://grafana.com/api/dashboards/12356/images/8223/image)

- Use dashboard_id `12356` or [read more how to import](https://grafana.com/grafana/dashboards/12356)
