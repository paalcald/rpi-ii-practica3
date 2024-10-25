---
title: Práctica 3. Servidores REST y representación de la información. JSON y CBOR
author: Pablo C. Alcalde y Alejandro de Celis
geometry: margin=2cm
papersize: a4
---
# Tarea 3.1

Los datos se codifican en JSON, siendo la estructura de los datos la siguiente:

```js
temperatura = {
    "raw": value
}

led = {
    "red": red_value,
    "green": green_value,
    "blue": blue_value,
}
```

Los datos no se encriptan de ninguna manera y estan disponibles para cualquiera que lea los paquetes.

# Tarea 3.2

Como la siguiente captura muestra, el trafico es efectivamente el mismo que el del cliente web.
Nos percatamos de un detalle curioso, pese a que si accedemos un endpoint inexistente el servidor reacciona de manera correcta respondiendo con `500 Internal Server Error`, cuando enviamos un paquete mal formado como puede ser que le falten miembros en el json, el servidor encuentra un error y reinicia, un comportamiento fatídico. De la misma manera, no se realiza ninguna comprobación sobre si los valores recibidos son correctos desde el servidor.

# Tarea 3.3
Hemos ignorado temporalmente todo lo relacionado con Node-RED.

# Tarea Entregable 1
Definimos las funciones necesarias para la transformación y generamos un handler y lo registramos.
```c
...

float to_fahrenheit(float celsius)
{
    return (1.8 * celsius + 32.0);
}

static esp_err_t temperature_f_data_get_handler(httpd_req_t *req)
{
    httpd_resp_set_type(req, "application/json");
    cJSON *root = cJSON_CreateObject();
    cJSON_AddNumberToObject(root, "fahrenheit",
                            (int) to_fahrenheit((float) (esp_random() % 20)));
    const char *sys_info = cJSON_Print(root);
    httpd_resp_sendstr(req, sys_info);
    free((void *)sys_info);
    cJSON_Delete(root);
    return ESP_OK;
}
...

    /* URI handler for fetching temperature data in fahrenheit*/
    httpd_uri_t temperature_f_get_uri = {
        .uri = "/api/v1/temp/fahrenheit",
        .method = HTTP_GET,
        .handler = temperature_f_data_get_handler,
        .user_ctx = rest_context
    };

    httpd_register_uri_handler(server, &temperature_f_get_uri);

```

# Tarea Entregable 2

La extensión no requiere más que eliminar el cast a `int` ya que, es la extracción la que difiere si extraemos un double o un float i.e. accediendo `valueint` o `valuedouble`; sin embargo la inserción es siempre con la misma función i.e `cJSON_AddNumberToObject`.

# Tarea Entregable 3

Vamos a mandar una JSON con la siguiente estructura para guardar un empleado en una base de datos situada en un nodo fog de nuestra red.
```js
{
    "name" : "John Doe",
    "salary" : 1200.0,
    "birthdate" : {
        "day" : 1,
        "month" : 1,
        "year" : 2000
    }
}
```
Nos parece bastante completo, ya que tiene nesting, strings, doubles e ints.

El siguiente handler se encarga de ello

```c
static esp_err_t employee_post_handler(httpd_req_t *req)
{
    int ret = 0;
    int total_len = req->content_len;
    int cur_len = 0;
    char *buf = ((rest_server_context_t *)(req->user_ctx))->scratch;
    int received = 0;
    if (total_len >= SCRATCH_BUFSIZE) {
        /* Respond with 500 Internal Server Error */
        httpd_resp_send_err(req, HTTPD_500_INTERNAL_SERVER_ERROR, "content too long");
        return ESP_FAIL;
    }
    while (cur_len < total_len) {
        received = httpd_req_recv(req, buf + cur_len, total_len);
        if (received <= 0) {
            /* Respond with 500 Internal Server Error */
            httpd_resp_send_err(req, HTTPD_500_INTERNAL_SERVER_ERROR,
                                "Failed to post control value");
            return ESP_FAIL;
        }
        cur_len += received;
    }
    buf[total_len] = '\0';

    cJSON *root = cJSON_Parse(buf);
    char *name = calloc(256, sizeof(char));
    cJSON *name_json = cJSON_GetObjectItem(root, "name");
    if (cJSON_IsString(name_json)) {
        strncpy(name, name_json->valuestring, 256);
        name[255] = '\0';
    } else {
        httpd_resp_send_err(req, HTTPD_400_BAD_REQUEST, "Wrong format");
        ret = -1;
        goto cleanup;
    }
    double salary = 0;
    cJSON *salary_json= cJSON_GetObjectItem(root, "salary");
    if (cJSON_IsNumber(salary_json)) {
        salary = salary_json->valuedouble;
    } else {
        httpd_resp_send_err(req, HTTPD_400_BAD_REQUEST, "Wrong format");
        ret = -1;
        goto cleanup;
    }
    int day = 0;
    int month = 0;
    int year = 0;
    cJSON *birthdate_json = cJSON_GetObjectItem(root, "birthdate");
    if (cJSON_IsObject(birthdate_json)) {
        cJSON *day_json = cJSON_GetObjectItem(birthdate_json, "day");
        if (cJSON_IsNumber(day_json)) {
            day = day_json->valueint;
        } else {
            httpd_resp_send_err(req, HTTPD_400_BAD_REQUEST, "Wrong day format");
            ret = -1;
            goto cleanup;
        }
        cJSON *month_json = cJSON_GetObjectItem(birthdate_json, "month");
        if (cJSON_IsNumber(month_json)) {
            month = month_json->valueint;
        } else {
            httpd_resp_send_err(req, HTTPD_400_BAD_REQUEST, "Wrong month format");
            ret = -1;
            goto cleanup;
        }
        cJSON *year_json = cJSON_GetObjectItem(birthdate_json, "year");
        if (cJSON_IsNumber(year_json)) {
            year = year_json->valueint;
        } else {
            httpd_resp_send_err(req, HTTPD_400_BAD_REQUEST, "Wrong year format");
            ret = -1;
            goto cleanup;
        }
    } else {
        httpd_resp_send_err(req, HTTPD_400_BAD_REQUEST, "Wrong birthdate format");
        ret = -1;
        goto cleanup;
    }
    ESP_LOGI(REST_TAG,
             "Received Employee: name = %s, salary = %.2f, birthdate = %d/%d/%d",
             name, salary, day, month, year);
    httpd_resp_sendstr(req, "Request to add employee received");
    goto cleanup;

cleanup:
    cJSON_Delete(root);
    free(name);
    return ret ? ESP_FAIL : ESP_OK;
}
```
Tambien registramos el handler aunque no se ha representado aquí por ser exáctamente igual que el caso del led.
Nos encargamos de hacerlo resilente a errores y capáz de devolver el código correcto si la petición no es válida.

Un dato curioso con el que nos enfrentamos mientras que estabamos gestionando los erroes es que la funcion `cJSON_Delete` elimina no solo los nodos de manera descendiente empezando en ese mismo, si no que también se encarga de la liberación del nodo padre, es decir, si intentasemos liberar cualquiera de los nodos hijos de `root`, como por ejemplo `day_json`. Incurririamos en una doble liberación.

Se adjuntan las [capturas](./rest-json-cbor/resful_server_entrega_3.pcapng)

# Tarea Entregable 4

Utilizaremos el componente `cbor` que siguiendo las instrucciones del [registro de componentes](https://components.espressif.com/components/espressif/cbor) se puede añadir a nuestro proyecto ejecutando el siguiente código:

```sh
idf.py add-dependency "espressif/cbor^0.6.0~1"
```

En primer lugar, para generar este último endpoint vamos a crear un complemento para agrupar toda la lógica referente a nuestro objeto. En este caso el componente es `employee_utils`.

```
components
└── employee_utils
    ├── CMakeLists.txt
    ├── employee_utils.c
    └── include
        └── employee_utils.h
```

`CMakeLists.txt`:

```cmake
idf_component_register(SRCS "employee_utils.c"
    INCLUDE_DIRS "include"
    REQUIRES cbor)
```

`employee_utils.h`:

```c
#ifndef EMPLOYEE_UTILS_H
#define EMPLOYEE_UTILS_H

#include "cbor.h"

#define CBOR_MAP_ERR(a, goto_tag) \
    if ((a) != CborNoError)       \
    {                             \
        ret = a;                  \
        goto goto_tag;            \
    }

struct birthdate {
    int day;
    int month;
    int year;
};

struct employee {
    char name[256];
    double salary;
    struct birthdate birthdate;
};

size_t employee_toCBOR(const struct employee *e, uint8_t *buf, size_t len);
size_t employee_toJSON(const struct employee *e, char* buf, size_t len);
CborError cbor_encode_birthdate(CborEncoder * it, const struct birthdate *b);
CborError cbor_envode_employee(CborEncoder * it, const struct employee *e);
CborError cbor_value_get_employee(CborValue *it, struct employee *emp);
CborError cbor_value_get_birthdate(CborValue *it, struct birthdate *bdate);
CborError cbor_value_get_employee(CborValue *it, struct employee *emp);
#endif // EMPLOYEE_UTILS_H
```

`employee_utils.c`:

```c
#include "employee_utils.h"
#include "esp_log.h"

static const char *TAG = "UTILS";

CborError cbor_value_get_birthdate(CborValue *it, struct birthdate *bdate)
{
    CborError ret;
    if (!cbor_value_is_map(it)) {
        ret = CborErrorImproperValue;
        goto err;
    }
    CborValue cbor_day;
    CBOR_MAP_ERR(cbor_value_map_find_value(it, "day", &cbor_day), err);
    CBOR_MAP_ERR(cbor_value_get_int(&cbor_day, &(bdate->day)), err);
    CborValue cbor_month;
    CBOR_MAP_ERR(cbor_value_map_find_value(it, "month", &cbor_month), err);
    CBOR_MAP_ERR(cbor_value_get_int(&cbor_month, &(bdate->month)), err);
    CborValue cbor_year;
    CBOR_MAP_ERR(cbor_value_map_find_value(it, "year", &cbor_year), err);
    CBOR_MAP_ERR(cbor_value_get_int(&cbor_year, &(bdate->year)), err);
    return CborNoError;
err:
    return ret;
}


CborError cbor_value_get_employee(CborValue *it, struct employee *emp)
{
    CborError ret;
    if (!cbor_value_is_map(it)) {
        return CborErrorImproperValue;
    }
    // Get the employee name
    CborValue name_it;
    CBOR_MAP_ERR(cbor_value_map_find_value(it, "name", &name_it), err);
    size_t name_len;
    CBOR_MAP_ERR(cbor_value_get_string_length(&name_it, &name_len), err);
    CBOR_MAP_ERR(cbor_value_copy_text_string(&name_it, (emp->name), &name_len, NULL), err);
    // Get the employee salary
    CborValue salary_it;
    CBOR_MAP_ERR(cbor_value_map_find_value(it, "salary", &salary_it), err);
    CBOR_MAP_ERR(cbor_value_get_double(&salary_it, &(emp->salary)), err);
    //Get the employee birthdate
    CborValue birthdate_it;
    CBOR_MAP_ERR(cbor_value_map_find_value(it, "birthdate", &birthdate_it), err);
    CBOR_MAP_ERR(cbor_value_get_birthdate(&birthdate_it, &(emp->birthdate)), err);
    cbor_value_advance(it);
    return CborNoError;
err:
    return ret;
}

size_t employee_toJSON(const struct employee *e, char *buf, size_t len)
{
    return snprintf(buf, len,
            "{\"name\":\"%s\",\"salary\":%.2f,\"birthdate\":{\"day\":%d,\"month\":%d,\"year\":%d}}",
            e->name, e->salary, e->birthdate.day, e->birthdate.month, e->birthdate.year);
}

CborError cbor_encode_birthdate(CborEncoder * it, const struct birthdate *b)
{
    CborError ret;
    CborEncoder attr;
    CBOR_MAP_ERR(cbor_encoder_create_map(it, &attr, 3), err);
    CBOR_MAP_ERR(cbor_encode_text_stringz(&attr, "day"), err);
    CBOR_MAP_ERR(cbor_encode_uint(&attr, b->day), err);
    CBOR_MAP_ERR(cbor_encode_text_stringz(&attr, "month"), err);
    CBOR_MAP_ERR(cbor_encode_uint(&attr, b->month), err);
    CBOR_MAP_ERR(cbor_encode_text_stringz(&attr, "year"), err);
    CBOR_MAP_ERR(cbor_encode_uint(&attr, b->year), err);
    CBOR_MAP_ERR(cbor_encoder_close_container(it, &attr), err);
    ESP_LOGI(TAG, "Birthdate encoded");
    return CborNoError;
err:
    return ret;
}

CborError cbor_encode_employee(CborEncoder *it, const struct employee *e)
{
    CborError ret;
    CborEncoder attr;
    CBOR_MAP_ERR(cbor_encoder_create_map(it, &attr, 3), err);
    CBOR_MAP_ERR(cbor_encode_text_stringz(&attr, "name"), err);
    CBOR_MAP_ERR(cbor_encode_text_stringz(&attr, e->name ), err);
    CBOR_MAP_ERR(cbor_encode_text_stringz(&attr, "salary"), err);
    CBOR_MAP_ERR(cbor_encode_double(&attr, e->salary), err);
    CBOR_MAP_ERR(cbor_encode_text_stringz(&attr, "birthdate"), err);
    CBOR_MAP_ERR(cbor_encode_birthdate(&attr, &(e->birthdate)), err);
    CBOR_MAP_ERR(cbor_encoder_close_container(it, &attr), err);
    ESP_LOGI(TAG, "Employee encoded");
    return CborNoError;
err:
    return ret;
}

size_t employee_toCBOR(const struct employee *e, uint8_t *buf, size_t len)
{
    ESP_LOGI(TAG, "attempting to encode employee");
    CborEncoder employee_encoder;
    cbor_encoder_init(&employee_encoder, buf, len, 0);
    cbor_encode_employee(&employee_encoder, e);
    return cbor_encoder_get_buffer_size(&employee_encoder, buf);
}
```

Con este trabajo previo la creación de los endpoints de post y de get se simplifica enormemente.

Generaremos dos endopoints, uno de `GET` y otro de `POST` por completitud. Para justificarlo un poco, el de `GET` nos podría devolver un ejemplo de un empleado. Para que así el cliente pueda enviarnos la codificación correcta de otros empleados.

`POST`:

```c
static esp_err_t employee_cbor_post_handler(httpd_req_t *req)
{
    int total_len = req->content_len;
    int cur_len = 0;
    char *buf = ((rest_server_context_t *)(req->user_ctx))->scratch;
    int received = 0;
    if (total_len >= SCRATCH_BUFSIZE) {
        /* Respond with 500 Internal Server Error */
        httpd_resp_send_err(req, HTTPD_500_INTERNAL_SERVER_ERROR, "content too long");
        return ESP_FAIL;
    }
    while (cur_len < total_len) {
        received = httpd_req_recv(req, buf + cur_len, total_len);
        if (received <= 0) {
            /* Respond with 500 Internal Server Error */
            httpd_resp_send_err(req, HTTPD_500_INTERNAL_SERVER_ERROR, "Failed to post control value");
            return ESP_FAIL;
        }
        cur_len += received;
    }
    buf[total_len] = '\0';

    CborParser root_parser;
    CborValue it;
    cbor_parser_init((uint8_t*) buf, total_len, 0, &root_parser, &it);
    struct employee employee;
    CborError ret = cbor_value_get_employee(&it, &employee);
    char employee_json[256];
    employee_toJSON(&employee, employee_json, 256);
    ESP_LOGI(REST_TAG, "Received employee:\n %s", employee_json);
    httpd_resp_sendstr(req, "Request to add employee received");
    return (ret == CborNoError) ? ESP_OK : ESP_FAIL;
}
```

`GET`:

```c
static esp_err_t example_employee_cbor_get_handler(httpd_req_t *req)
{
    httpd_resp_set_type(req, "application/cbor");
    struct employee example_emp;
    strcpy(example_emp.name, "John Doe");
    example_emp.salary = 1200.0;
    example_emp.birthdate.day = 1;
    example_emp.birthdate.month = 1;
    example_emp.birthdate.year = 2000;
    uint8_t buf[256];
    size_t len = employee_toCBOR(&example_emp, buf, 256);
    httpd_resp_send(req, (char*) buf, len);
    return ESP_OK;
}
```

Finalmente esto junto con la información asociada nos permite hacer el envio y la recepción de datos como puede observarse en la siguiente [captura](./resful_server_entrega_5.pcapng). Como anotación, nos encontramos con problemas a la hora de enviar la información al servidor ya que para enviar datos binarios tenemos que utilizar el siguiente comando.

```sh
curl --data-binary @employee.cbor -H "Content-Type: application/cbor" -X POST http://192.168.1.22/api/v2/employee/add
```

