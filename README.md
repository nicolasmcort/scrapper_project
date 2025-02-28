# scrapper_project

``` python
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import Select
import time



# Inicializar el driver (Edge en este caso)
driver = webdriver.Edge()

# 1. Abrir el sitio web
url = "https://sia.unal.edu.co/Catalogo/facespublico/public/servicioPublico.jsf;PortalJSESSION=fMdOk3jZADUC9bEclL_BFOwAxNoviHdXCcxAw4QBrb8TXl2tXUkB!498583482?taskflowId=task-flow-AC_CatalogoAsignaturas"
driver.get(url)

# Lista de códigos de materia y créditos
codigos_materias = ["2016699", "2016707"]
creditos = [3, 3]

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
    input_element = driver.find_element(By.ID, id_input)
    input_element.clear()  # Borra cualquier texto previo en el campo
    input_element.send_keys(texto)
    time.sleep(1)  # Pausa para evitar errores

# 5. Función para hacer clic en un botón
def hacer_clic_boton(id_boton):
    """Encuentra un botón por su ID y hace clic en él."""
    boton = driver.find_element(By.ID, id_boton)
    boton.click()
    time.sleep(3)  # Esperar para ver los cambios

# Iterar sobre cada código de materia y su respectivo número de créditos
for codigo_materia, credito in zip(codigos_materias, creditos):
    # Ingresar el número de créditos en el campo "Nombre de la asignatura"
    ingresar_texto("pt1:r1:0:it10::content", str(credito))
    
    # Hacer clic en el botón "Mostrar"
    hacer_clic_boton("pt1:r1:0:cb1")
    
    try:
        # Buscar el enlace dentro de la tabla usando XPath
        enlace = driver.find_element(By.XPATH, f"//a[contains(text(), '{codigo_materia}')]")
        enlace.click()
        print(f"Se hizo clic en la materia {codigo_materia}")
        time.sleep(5)  # Esperar a que la información cargue
        
        # Extraer todo el texto de la página
        texto_pagina = driver.find_element(By.TAG_NAME, "body").text
        print("Texto extraído de la página:")
        print(texto_pagina)
        
        # Hacer clic en el botón "Volver"
        boton_volver = driver.find_element(By.ID, "pt1:r1:1:cb4")
        boton_volver.click()
        print("Se hizo clic en el botón 'Volver'")
        
        time.sleep(3)  # Esperar a que la página regrese correctamente
    
    except Exception as e:
        print(f"No se encontró la materia con código {codigo_materia}: {e}")
    
# Cerrar el navegador
driver.quit()

```
