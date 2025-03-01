# scrapper_project

``` python
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import Select, WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
import time

# Inicializar el driver (Edge en este caso)
driver = webdriver.Edge()

# 1. Abrir el sitio web
url = "https://sia.unal.edu.co/Catalogo/facespublico/public/servicioPublico.jsf;PortalJSESSION=fMdOk3jZADUC9bEclL_BFOwAxNoviHdXCcxAw4QBrb8TXl2tXUkB!498583482?taskflowId=task-flow-AC_CatalogoAsignaturas"
driver.get(url)

# Lista de códigos de materia y créditos
codigos_materias = ["2016707", "2016699", "2015174", "2016696"]
creditos = [3, 3, 4, 3]

# 2. Esperar a que la página cargue
time.sleep(5)

# 3. Llenar los desplegables
def seleccionar_opcion_por_texto(id_select, texto):
    """Encuentra un <select> por su ID y selecciona la opción con el texto dado."""
    select_element = driver.find_element(By.ID, id_select)
    select = Select(select_element)
    select.select_by_visible_text(texto)
    time.sleep(1)  # Pequeña pausa para evitar errores

# Seleccionar "Pregrado"
seleccionar_opcion_por_texto("pt1:r1:0:soc1::content", "Pregrado")

# Seleccionar "1101 SEDE BOGOTÁ"
seleccionar_opcion_por_texto("pt1:r1:0:soc9::content", "1101 SEDE BOGOTÁ")

# Seleccionar "2055 FACULTAD DE INGENIERÍA"
seleccionar_opcion_por_texto("pt1:r1:0:soc2::content", "2055 FACULTAD DE INGENIERÍA")

# Seleccionar "2A74 INGENIERÍA DE SISTEMAS Y COMPUTACIÓN"
seleccionar_opcion_por_texto("pt1:r1:0:soc3::content", "2A74 INGENIERÍA DE SISTEMAS Y COMPUTACIÓN")

# 4. Función para ingresar texto en un campo
def ingresar_texto(id_input, texto):
    """Encuentra un <input> por su ID y escribe el texto dado."""
    try:
        # Esperar a que el campo de texto esté presente
        input_element = WebDriverWait(driver, 20).until(
            EC.presence_of_element_located((By.ID, id_input))
        )
        input_element.clear()  # Borra cualquier texto previo en el campo
        input_element.send_keys(texto)
        time.sleep(1)  # Pausa para evitar errores
    except Exception as e:
        print(f"No se pudo encontrar el campo de texto con ID {id_input}: {e}")
        raise  # Relanzar la excepción para detener la ejecución

# 5. Función para hacer clic en un botón
def hacer_clic_boton(id_boton):
    """Encuentra un botón por su ID y hace clic en él."""
    try:
        boton = WebDriverWait(driver, 20).until(
            EC.element_to_be_clickable((By.ID, id_boton))
        )
        boton.click()
        time.sleep(3)  # Esperar para ver los cambios
    except Exception as e:
        print(f"No se pudo hacer clic en el botón con ID {id_boton}: {e}")
        raise  # Relanzar la excepción para detener la ejecución

# 6. Función para volver a la página principal
def volver_a_pagina_principal():
    """Hace clic en el botón 'Volver' y verifica que la página haya regresado correctamente."""
    try:
        # Buscar el botón "Volver" usando un XPath genérico
        boton_volver = WebDriverWait(driver, 20).until(
            EC.element_to_be_clickable((By.XPATH, "//div[contains(@id, 'cb4')]//a[contains(@class, 'af_button_link') and contains(.//span, 'Volver')]"))
        )
        
        # Desplazarse hasta el botón "Volver" si es necesario
        driver.execute_script("arguments[0].scrollIntoView(true);", boton_volver)
        time.sleep(1)  # Esperar a que el botón esté en la vista
        
        # Hacer clic en el botón "Volver"
        boton_volver.click()
        print("Se hizo clic en el botón 'Volver'")
        time.sleep(3)  # Esperar a que la página regrese correctamente

        # Verificar si la página ha regresado correctamente
        WebDriverWait(driver, 20).until(
            EC.presence_of_element_located((By.ID, "pt1:r1:0:it10::content"))
        )
    except Exception as e:
        print(f"No se pudo regresar a la página principal: {e}")
        # Recargar la página manualmente si no se pudo regresar
        driver.get(url)
        time.sleep(5)  # Esperar a que la página cargue
        # Volver a seleccionar las opciones necesarias
        seleccionar_opcion_por_texto("pt1:r1:0:soc1::content", "Pregrado")
        seleccionar_opcion_por_texto("pt1:r1:0:soc9::content", "1101 SEDE BOGOTÁ")
        seleccionar_opcion_por_texto("pt1:r1:0:soc2::content", "2055 FACULTAD DE INGENIERÍA")
        seleccionar_opcion_por_texto("pt1:r1:0:soc3::content", "2A74 INGENIERÍA DE SISTEMAS Y COMPUTACIÓN")

# Variable para almacenar el último número de créditos ingresado
ultimo_credito = None

# Iterar sobre cada código de materia y su respectivo número de créditos
for codigo_materia, credito in zip(codigos_materias, creditos):
    try:
        # Ingresar el número de créditos solo si es diferente al último ingresado
        if credito != ultimo_credito:
            ingresar_texto("pt1:r1:0:it10::content", str(credito))
            ultimo_credito = credito  # Actualizar el último crédito ingresado
            
            # Hacer clic en el botón "Mostrar" solo si se modificaron los créditos
            hacer_clic_boton("pt1:r1:0:cb1")
        
        # Esperar a que la tabla cargue usando WebDriverWait
        WebDriverWait(driver, 20).until(
            EC.presence_of_element_located((By.XPATH, f"//a[contains(text(), '{codigo_materia}')]"))
        )
        
        # Esperar a que el enlace de la materia sea clickeable
        enlace = WebDriverWait(driver, 20).until(
            EC.element_to_be_clickable((By.XPATH, f"//a[contains(text(), '{codigo_materia}')]"))
        )
        enlace.click()
        print(f"Se hizo clic en la materia {codigo_materia}")
        time.sleep(5)  # Esperar a que la información cargue
        
        # Extraer todo el texto de la página
        texto_pagina = driver.find_element(By.TAG_NAME, "body").text
        print("Texto extraído de la página:")
        print(texto_pagina)
        
        # Volver a la página principal
        volver_a_pagina_principal()
    
    except Exception as e:
        print(f"No se pudo procesar la materia con código {codigo_materia}: {e}")
        # Recargar la página manualmente si hay un error
        driver.get(url)
        time.sleep(5)  # Esperar a que la página cargue
        # Volver a seleccionar las opciones necesarias
        seleccionar_opcion_por_texto("pt1:r1:0:soc1::content", "Pregrado")
        seleccionar_opcion_por_texto("pt1:r1:0:soc9::content", "1101 SEDE BOGOTÁ")
        seleccionar_opcion_por_texto("pt1:r1:0:soc2::content", "2055 FACULTAD DE INGENIERÍA")
        seleccionar_opcion_por_texto("pt1:r1:0:soc3::content", "2A74 INGENIERÍA DE SISTEMAS Y COMPUTACIÓN")
    
# Cerrar el navegador
driver.quit()
```
