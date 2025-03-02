# scrapper_project

***
### Código de la lógica del proyecto (Sin GUI)

```python
import heapq
from typing import List, Tuple, Dict, Any
import re
import json
import os
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import Select, WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
import time


# Clase para representar una materia
class Subject:
    def __init__(self, name: str, group: int, professor: str, schedule: Tuple[str, str], days: List[str], preference: int = 0) -> None:
        self.name = name
        self.group = group
        self.professor = professor
        self.schedule = schedule  # (inicio, fin)
        self.days = days  # Lista de días (ejemplo: ['LUNES', 'MIÉRCOLES'])
        self.preference = preference  # Preferencia de la materia

    def __lt__(self, other):
        """
        Define cómo se comparan dos materias. Se usa para ordenar en el heap.
        """
        return self.preference < other.preference

    def __str__(self) -> str:
        """Representación en cadena de la materia."""
        return f"Materia: {self.name} \n" \
               f"\tProfesor: {self.professor} \n" \
               f"\tGrupo: {self.group} \n" \
               f"\tHorario: {self.schedule[0]} - {self.schedule[1]}\n" \
               f"\tDías: {', '.join(self.days)}\n" \
               f"\tPreferencia: {self.preference}"

# Clase para organizar horarios y evitar colisiones
class ScheduleOrganizer:
    @staticmethod
    def has_conflict(subject: Subject, selected_subjects: List[Subject]) -> bool:
        """
        Verifica si una materia entra en conflicto con las ya seleccionadas.

        :param subject: Materia a verificar.
        :param selected_subjects: Lista de materias ya seleccionadas.
        :return: True si hay conflicto, False en caso contrario.
        """
        for selected in selected_subjects:
            # Verificar si hay al menos un día en común
            if set(subject.days) & set(selected.days):
                # Verificar si los horarios se solapan
                start1, end1 = subject.schedule
                start2, end2 = selected.schedule
                if not (end1 <= start2 or end2 <= start1):
                    return True  # Hay conflicto
        return False


# Clase para generar combinaciones de horarios usando un heap
class CombinationGenerator:
    @staticmethod
    def generate_combinations(subjects: List[List[Subject]], limit: int) -> List[List[Subject]]:
        """
        Genera combinaciones válidas de horarios usando un heap.

        :param subjects: Lista de listas de materias, donde cada lista interna representa una materia y sus grupos.
        :param limit: Límite de combinaciones a generar.
        :return: Lista de combinaciones válidas.
        """
        combinations = []  # Lista para almacenar las combinaciones válidas
        heap = []  # Heap para priorizar las combinaciones por preferencia acumulada

        # Inicializar el heap con la raíz (sin materias seleccionadas)
        heapq.heappush(heap, (-0, [], 0))  # (preferencia acumulada negativa, combinación actual, nivel)

        while heap and len(combinations) < limit:
            # Extraer la combinación con mayor preferencia acumulada
            neg_accumulated_preference, current_combination, level = heapq.heappop(heap)
            accumulated_preference = -neg_accumulated_preference  # Convertir a positivo

            # Si ya se han procesado todas las materias, agregar la combinación a la lista
            if level == len(subjects):
                combinations.append((accumulated_preference, current_combination))
                continue

            # Obtener la materia actual y sus grupos
            current_subject_groups = subjects[level]

            # Ordenar los grupos por preferencia (de mayor a menor)
            sorted_groups = sorted(current_subject_groups, key=lambda x: x.preference, reverse=True)

            # Probar agregar cada grupo de la materia actual
            for subject in sorted_groups:
                # Verificar si la materia entra en conflicto con las ya seleccionadas
                if not ScheduleOrganizer.has_conflict(subject, current_combination):
                    # Calcular la nueva preferencia acumulada
                    new_accumulated_preference = accumulated_preference + subject.preference

                    # Crear una nueva combinación
                    new_combination = current_combination + [subject]

                    # Insertar la nueva combinación en el heap
                    heapq.heappush(heap, (-new_accumulated_preference, new_combination, level + 1))

        # Ordenar las combinaciones finales por preferencia total (de mayor a menor)
        combinations.sort(reverse=True, key=lambda x: x[0])

        # Devolver solo las combinaciones (sin la preferencia acumulada)
        return [combo for (_, combo) in combinations]


# Clase para representar un usuario con sus datos y preferencias
class Usuario:
    def __init__(self):
        """
        Representa un usuario con sus datos y preferencias.
        """
        # Valores predefinidos
        self.nivel_de_estudios = "Pregrado"
        self.sede = "1101 SEDE BOGOTÁ"
        self.facultad = "2055 FACULTAD DE INGENIERÍA"
        self.plan_de_estudios = None
        self.navegador = None  # Nuevo campo para el navegador
        self.preferencias_generales = {}
        self.pesos = {}  # Pesos asignados a los criterios
        self.limit = 8  # Límite de horarios a generar (default 8)
        self.grupos_favoritos = {}  # Grupos favoritos por materia

    def ingresar_datos(self):
        """
        Solicita al usuario que ingrese sus datos y preferencias.
        """
        # El usuario ingresa el plan de estudios y el navegador
        self.plan_de_estudios = input("Ingrese el plan de estudios: ")
        self.navegador = input("Ingrese el navegador que usa (Chrome/Edge/Firefox): ").strip().capitalize()

        # Ingresar preferencias generales
        print("\nIngrese sus preferencias generales:")
        self.preferencias_generales = {
            "horario": input("Preferencia de horario (mañana/tarde): ").strip().lower(),
            "dias": input("Preferencia de días (ejemplo: L,M,J): ").strip().split(",")
        }

        # Asignar pesos a los criterios
        self.asignar_pesos()

        # Ingresar el límite de horarios a generar
        self.limit = int(input("Ingrese el límite de horarios a generar (default 8): ") or 8)

    def asignar_pesos(self):
        """
        Asigna pesos a los criterios en función de la importancia seleccionada por el usuario.
        """
        print("\nAsigne la importancia de cada criterio (poco/normal/bastante):")
        for criterio in ["grupo", "horario", "dias"]:
            importancia = input(f"Importancia de '{criterio}' (poco/normal/bastante): ").strip().lower()
            if importancia == "poco":
                self.pesos[criterio] = 1
            elif importancia == "normal":
                self.pesos[criterio] = 2
            elif importancia == "bastante":
                self.pesos[criterio] = 3
            else:
                print(f"Valor no válido para '{criterio}'. Se asignará peso 'normal'.")
                self.pesos[criterio] = 2

    def ingresar_grupos_favoritos(self, materias: List[str]):
        """
        Solicita al usuario que ingrese sus grupos favoritos para cada materia.

        :param materias: Lista de nombres de materias.
        """
        print("\nIngrese sus grupos favoritos para cada materia:")
        for materia in materias:
            grupos = input(f"Grupos favoritos para {materia} (ejemplo: 1,3): ").strip().split(",")
            self.grupos_favoritos[materia] = [int(grupo.strip()) for grupo in grupos]

    def to_dict(self) -> Dict[str, Any]:
        """
        Convierte los datos del usuario a un diccionario.

        :return: Diccionario con los datos del usuario.
        """
        return {
            "nivel_de_estudios": self.nivel_de_estudios,
            "sede": self.sede,
            "facultad": self.facultad,
            "plan_de_estudios": self.plan_de_estudios,
            "navegador": self.navegador,  # Nuevo campo
            "preferencias_generales": self.preferencias_generales,
            "pesos": self.pesos,  # Guardar los pesos en el JSON
            "limit": self.limit  # Guardar el límite en el JSON
        }

    @classmethod
    def from_dict(cls, datos: Dict[str, Any]):
        """
        Crea una instancia de Usuario a partir de un diccionario.

        :param datos: Diccionario con los datos del usuario.
        :return: Instancia de Usuario.
        """
        usuario = cls()
        usuario.plan_de_estudios = datos["plan_de_estudios"]
        usuario.navegador = datos.get("navegador", "Desconocido")  # Cargar el navegador
        usuario.preferencias_generales = datos["preferencias_generales"]
        usuario.pesos = datos.get("pesos", {})  # Cargar los pesos desde el JSON
        usuario.limit = datos.get("limit", 8)  # Cargar el límite desde el JSON (default 8)
        return usuario


# Clase para gestionar archivos JSON
class GestorJSON:
    def __init__(self, carpeta: str):
        """
        Gestiona la creación y guardado de archivos JSON.

        :param carpeta: Ruta de la carpeta donde se guardarán los archivos.
        """
        self.carpeta = carpeta

    def guardar_datos(self, datos: Dict[str, Any], nombre_archivo: str):
        """
        Guarda los datos en un archivo JSON dentro de la carpeta especificada.

        :param datos: Diccionario con los datos a guardar.
        :param nombre_archivo: Nombre del archivo JSON.
        """
        # Crear la carpeta si no existe
        if not os.path.exists(self.carpeta):
            os.makedirs(self.carpeta)

        # Ruta completa del archivo (usando os.path.join para evitar problemas con barras)
        ruta_archivo = os.path.join(self.carpeta, nombre_archivo)

        # Guardar los datos en el archivo JSON
        with open(ruta_archivo, "w", encoding="utf-8") as archivo:
            json.dump(datos, archivo, indent=4, ensure_ascii=False)

        print(f"Datos guardados en {ruta_archivo}")

    def cargar_datos(self, nombre_archivo: str) -> Dict[str, Any]:
        """
        Carga los datos desde un archivo JSON.

        :param nombre_archivo: Nombre del archivo JSON.
        :return: Diccionario con los datos cargados.
        """
        # Ruta completa del archivo
        ruta_archivo = os.path.join(self.carpeta, nombre_archivo)

        # Cargar los datos desde el archivo JSON
        with open(ruta_archivo, "r", encoding="utf-8") as archivo:
            datos = json.load(archivo)

        return datos


# Clase para extraer materias del SIA
class SubjectScraper:
    def __init__(self, browser_name, nivel_de_estudios, sede, facultad, plan_de_estudios):
        self.driver = BrowserFactory.get_browser(browser_name)  # Inicializar el driver en modo headless
        self.url = "https://sia.unal.edu.co/Catalogo/facespublico/public/servicioPublico.jsf;PortalJSESSION=fMdOk3jZADUC9bEclL_BFOwAxNoviHdXCcxAw4QBrb8TXl2tXUkB!498583482?taskflowId=task-flow-AC_CatalogoAsignaturas"
        self.subject_codes = ["2016707", "2016699", "2016375"]  # Códigos de materias
        self.nivel_de_estudios = nivel_de_estudios
        self.sede = sede
        self.facultad = facultad
        self.plan_de_estudios = plan_de_estudios
        self.subjects = []  # Lista de materias extraídas

    def open_website(self):
        """Abre la URL del sitio web."""
        self.driver.get(self.url)
        # Esperar a que la página cargue completamente
        WebDriverWait(self.driver, 20).until(
            EC.presence_of_element_located((By.ID, "pt1:r1:0:soc1::content"))
        )

    def select_option_by_text(self, select_id, text):
        """Selecciona una opción en un <select> por su texto visible."""
        try:
            # Esperar a que el desplegable esté habilitado
            WebDriverWait(self.driver, 20).until(
                EC.element_to_be_clickable((By.ID, select_id))
            )
            select_element = self.driver.find_element(By.ID, select_id)
            select = Select(select_element)
            select.select_by_visible_text(text)
            time.sleep(1)  # Pequeña pausa para evitar errores
        except Exception as e:
            print(f"No se pudo seleccionar la opción '{text}' en el desplegable con ID {select_id}: {e}")
            raise

    def click_button(self, button_id):
        """Hace clic en un botón por su ID."""
        try:
            button = WebDriverWait(self.driver, 20).until(
                EC.element_to_be_clickable((By.ID, button_id)))
            button.click()
            time.sleep(3)  # Esperar para ver los cambios
        except Exception as e:
            print(f"No se pudo hacer clic en el botón con ID {button_id}: {e}")
            raise

    def return_to_main_page(self):
        """Vuelve a la página principal haciendo clic en el botón 'Volver'."""
        try:
            return_button = WebDriverWait(self.driver, 20).until(
                EC.element_to_be_clickable((By.XPATH, "//div[contains(@id, 'cb4')]//a[contains(@class, 'af_button_link') and contains(.//span, 'Volver')]"))
            )
            self.driver.execute_script("arguments[0].scrollIntoView(true);", return_button)
            time.sleep(1)
            return_button.click()
            print("Se hizo clic en el botón 'Volver'")
            time.sleep(3)

            # Verificar que la página ha regresado correctamente
            WebDriverWait(self.driver, 20).until(
                EC.presence_of_element_located((By.ID, "pt1:r1:0:it10::content"))
            )
        except Exception as e:
            print(f"No se pudo regresar a la página principal: {e}")
            self.reset_page()

    def reset_page(self):
        """Recarga la página y selecciona las opciones iniciales."""
        self.driver.get(self.url)
        time.sleep(5)
        self.select_initial_options()

    def select_initial_options(self):
        """Selecciona las opciones iniciales en el orden correcto."""
        self.select_option_by_text("pt1:r1:0:soc1::content", self.nivel_de_estudios)
        self.select_option_by_text("pt1:r1:0:soc9::content", self.sede)
        self.select_option_by_text("pt1:r1:0:soc2::content", self.facultad)
        self.select_option_by_text("pt1:r1:0:soc3::content", self.plan_de_estudios)

    def scrape_subject_info(self, subject_code):
        """Extrae la información de una materia específica."""
        try:
            # Esperar a que la tabla cargue
            WebDriverWait(self.driver, 20).until(
                EC.presence_of_element_located((By.XPATH, f"//a[contains(text(), '{subject_code}')]"))
            )
            # Hacer clic en el enlace de la materia
            link = WebDriverWait(self.driver, 20).until(
                EC.element_to_be_clickable((By.XPATH, f"//a[contains(text(), '{subject_code}')]"))
            )
            link.click()
            print(f"Se hizo clic en la materia {subject_code}")
            time.sleep(5)

            # Extraer el texto de la página
            page_text = self.driver.find_element(By.TAG_NAME, "body").text

            # Parsear la información de la materia
            subjects = SubjectParser.parse_subject_info(page_text)
            self.subjects.extend(subjects)  # Agregar las materias extraídas a la lista

            # Volver a la página principal
            self.return_to_main_page()
        except Exception as e:
            print(f"No se pudo procesar la materia con código {subject_code}: {e}")
            self.reset_page()

    def run(self):
        """Ejecuta el proceso completo de scraping."""
        try:
            self.open_website()
            self.select_initial_options()

            # Hacer clic en "Mostrar" una sola vez para cargar todas las materias
            self.click_button("pt1:r1:0:cb1")

            # Iterar sobre cada materia
            for subject_code in self.subject_codes:
                self.scrape_subject_info(subject_code)

        finally:
            # Cerrar el navegador al finalizar
            self.driver.quit()


# Clase para parsear la información de las materias
class SubjectParser:
    @staticmethod
    def parse_subject_info(text: str) -> List[Subject]:
        subjects = []
        subject_name = re.search(r"Información de la asignatura\n\s+Volver\n(.+?)\n", text).group(1).strip()

        # Dividir el texto por la palabra "Grupo" y capturar el número de grupo
        group_matches = list(re.finditer(r"\((\d+)\) Grupo (\d+)\s+Profesor: (.+?)\.", text))
        for match in group_matches:
            group_number = int(match.group(2))  # Número de grupo
            professor = match.group(3).strip()  # Profesor

            # Extraer la sección correspondiente al grupo
            start = match.end()
            next_match = next((m for m in group_matches if m.start() > start), None)
            end = next_match.start() if next_match else len(text)
            section = text[start:end].strip()

            # Extraer los días de la semana (palabras en mayúscula)
            days = re.findall(r"\b(LUNES|MARTES|MIÉRCOLES|JUEVES|VIERNES|SÁBADO|DOMINGO)\b", section)

            # Extraer el horario (primera coincidencia)
            schedule_match = re.search(r"(\d{2}:\d{2}) a (\d{2}:\d{2})", section)
            schedule = (schedule_match.group(1), schedule_match.group(2)) if schedule_match else ("00:00", "00:00")

            # Crear instancia de Subject
            subject = Subject(
                name=subject_name,
                group=group_number,
                professor=professor,
                schedule=schedule,
                days=days
            )
            subjects.append(subject)

        return subjects


# Clase para crear instancias de navegadores
class BrowserFactory:
    """Factory para crear instancias de navegadores en modo headless."""
    @staticmethod
    def get_browser(browser_name):
        if browser_name.lower() == "chrome":
            options = webdriver.ChromeOptions()
            options.add_argument("--headless")  # Modo headless para Chrome
            return webdriver.Chrome(options=options)
        elif browser_name.lower() == "edge":
            options = webdriver.EdgeOptions()
            options.add_argument("--headless")  # Modo headless para Edge
            return webdriver.Edge(options=options)
        elif browser_name.lower() == "firefox":
            options = webdriver.FirefoxOptions()
            options.add_argument("--headless")  # Modo headless para Firefox
            return webdriver.Firefox(options=options)
        else:
            raise ValueError(f"Navegador no soportado: {browser_name}")


# Función para calcular la preferencia de una materia en función de las preferencias del usuario
def calcular_preferencia(subject: Subject, preferencias: Dict[str, Any], pesos: Dict[str, int], grupos_favoritos: Dict[str, List[int]]) -> int:
    """
    Calcula la preferencia de una materia en función de las preferencias del usuario.

    :param subject: Materia a evaluar.
    :param preferencias: Preferencias del usuario.
    :param pesos: Pesos asignados a los criterios.
    :param grupos_favoritos: Grupos favoritos del usuario.
    :return: Preferencia calculada.
    """
    preferencia = 0

    # Preferencia de horario
    hora_inicio = int(subject.schedule[0].split(':')[0])  # Extraer la hora y convertirla a entero
    if preferencias["horario"] == "mañana" and hora_inicio < 12:
        preferencia += 3 * pesos["horario"]
    elif preferencias["horario"] == "tarde" and hora_inicio >= 12:
        preferencia += 3 * pesos["horario"]

    # Preferencia de días
    dias_preferidos = set(preferencias["dias"])
    dias_materia = set(subject.days)
    if dias_preferidos & dias_materia:
        preferencia += 3 * pesos["dias"]

    # Preferencia de grupo favorito
    if subject.name in grupos_favoritos and subject.group in grupos_favoritos[subject.name]:
        preferencia += 3 * pesos["grupo"]

    return preferencia


# Función principal
def main():
    # Definir la ruta de la carpeta (usando barras normales)
    carpeta = "C:/Users/nicol/OneDrive - Universidad Nacional de Colombia/GENERAL/Documents/UNAL/Semestre 2/POO_JSON"
    archivo_usuario = "datos_usuario.json"

    # Crear una instancia de GestorJSON
    gestor_json = GestorJSON(carpeta=carpeta)

    # Verificar si ya existe un archivo de preferencias
    if os.path.exists(os.path.join(carpeta, archivo_usuario)):
        print("Cargando datos del usuario desde el archivo...")
        datos_usuario = gestor_json.cargar_datos(archivo_usuario)
        usuario = Usuario.from_dict(datos_usuario)

        # Verificar y solicitar datos faltantes
        if not usuario.plan_de_estudios:
            usuario.plan_de_estudios = input("Ingrese el plan de estudios: ")
        if not usuario.navegador:
            usuario.navegador = input("Ingrese el navegador que usa (Chrome/Edge/Firefox): ").strip().capitalize()
        if not usuario.preferencias_generales.get("horario"):
            usuario.preferencias_generales["horario"] = input("Preferencia de horario (mañana/tarde): ").strip().lower()
        if not usuario.preferencias_generales.get("dias"):
            usuario.preferencias_generales["dias"] = input("Preferencia de días (ejemplo: L,M,J): ").strip().split(",")
        if not usuario.pesos.get("grupo"):
            usuario.asignar_pesos()
        if not usuario.pesos.get("horario"):
            usuario.asignar_pesos()
        if not usuario.pesos.get("dias"):
            usuario.asignar_pesos()
        if not usuario.limit:
            usuario.limit = int(input("Ingrese el límite de horarios a generar (default 8): ") or 8)

        # Guardar los datos actualizados
        gestor_json.guardar_datos(usuario.to_dict(), archivo_usuario)
    else:
        print("No se encontró un archivo de preferencias. Ingrese sus datos:")
        usuario = Usuario()
        usuario.ingresar_datos()
        gestor_json.guardar_datos(usuario.to_dict(), archivo_usuario)

    # Mostrar los datos del usuario (incluyendo el navegador)
    print("\nDatos del usuario:")
    print(f"Nivel de estudios: {usuario.nivel_de_estudios}")
    print(f"Sede: {usuario.sede}")
    print(f"Facultad: {usuario.facultad}")
    print(f"Plan de estudios: {usuario.plan_de_estudios}")
    print(f"Navegador: {usuario.navegador}")
    print(f"Preferencias generales: {usuario.preferencias_generales}")
    print(f"Pesos: {usuario.pesos}")
    print(f"Límite de horarios a generar: {usuario.limit}")

    # Crear una instancia de SubjectScraper para extraer las materias
    scraper = SubjectScraper(usuario.navegador, usuario.nivel_de_estudios, usuario.sede, usuario.facultad, usuario.plan_de_estudios)
    scraper.run()

    # Obtener las materias extraídas
    materias_extraidas = scraper.subjects

    # Solicitar al usuario que ingrese sus grupos favoritos
    materias_unicas = list(set(subject.name for subject in materias_extraidas))  # Nombres únicos de materias
    usuario.ingresar_grupos_favoritos(materias_unicas)

    # Calcular la preferencia de cada materia en función de las preferencias del usuario
    for subject in materias_extraidas:
        subject.preference = calcular_preferencia(subject, usuario.preferencias_generales, usuario.pesos, usuario.grupos_favoritos)

    # Organizar las materias en una lista de listas (una lista por materia)
    materias_organizadas = {}
    for subject in materias_extraidas:
        if subject.name not in materias_organizadas:
            materias_organizadas[subject.name] = []
        materias_organizadas[subject.name].append(subject)

    # Convertir el diccionario a una lista de listas
    subjects = list(materias_organizadas.values())

    # Generar combinaciones válidas usando el heap
    valid_schedules = CombinationGenerator.generate_combinations(subjects, usuario.limit)

    # Mostrar los horarios generados
    print("\nHorarios válidos generados:")
    for i, schedule in enumerate(valid_schedules, start=1):
        print(f"\nHorario {i}:")
        for subject in schedule:
            print(subject)


if __name__ == "__main__":
    main()
```

***
### Código del scrapper
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
