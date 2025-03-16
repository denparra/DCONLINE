import streamlit as st
import sqlite3
import pandas as pd
import datetime
from io import BytesIO
import os

# --- Estilos y JavaScript ---
st.markdown("""
    <style>
    .stApp {
        max-width: 1200px;
        margin: auto;
    }
    input, textarea {
        font-size: 1.1em;
    }
    </style>
    """, unsafe_allow_html=True)

disable_enter_js = """
<script>
document.addEventListener('DOMContentLoaded', function() {
  const inputs = document.querySelectorAll('input');
  inputs.forEach(input => {
    input.addEventListener('keydown', function(e) {
      if (e.key === 'Enter') {
        e.preventDefault();
      }
    });
  });
});
</script>
"""

# --- Configuración de la Base de Datos ---
db_filename = 'datos_consignacion.db'

def get_connection():
    conn = sqlite3.connect(db_filename, check_same_thread=False)
    return conn

def create_tables(conn):
    cursor = conn.cursor()
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS links_contactos (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            link_general TEXT NOT NULL,
            fecha_creacion TEXT NOT NULL,
            marca TEXT NOT NULL,
            descripcion TEXT NOT NULL
        )
    ''')
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS contactos (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            link_auto TEXT NOT NULL,
            telefono TEXT UNIQUE NOT NULL,
            nombre TEXT NOT NULL,
            auto TEXT NOT NULL,
            precio REAL NOT NULL,
            descripcion TEXT NOT NULL,
            id_link INTEGER,
            FOREIGN KEY (id_link) REFERENCES links_contactos(id)
        )
    ''')
    conn.commit()

conn = get_connection()
create_tables(conn)

# --- Navegación en Streamlit con botones verticales ---
if 'page' not in st.session_state:
    st.session_state.page = "Crear Link Contactos"

st.sidebar.title("Navegación")
if st.sidebar.button("Crear Link Contactos"):
    st.session_state.page = "Crear Link Contactos"
if st.sidebar.button("Agregar Contactos"):
    st.session_state.page = "Agregar Contactos"
if st.sidebar.button("Ver Contactos & Exportar"):
    st.session_state.page = "Ver Contactos & Exportar"

page = st.session_state.page

# --- Página: Crear Link Contactos ---
if page == "Crear Link Contactos":
    st.title("Crear Link Contactos")
    st.markdown(disable_enter_js, unsafe_allow_html=True)
    with st.form("crear_link_form"):
        link_general = st.text_input("Link General")
        fecha_creacion = st.date_input("Fecha de Creación", value=datetime.date.today())
        marca = st.text_input("Marca")
        descripcion = st.text_area("Descripción")
        submitted = st.form_submit_button("Crear Link")
    if submitted:
        if not link_general.strip() or not marca.strip() or not descripcion.strip():
            st.error("Todos los campos son requeridos.")
        else:
            cursor = conn.cursor()
            cursor.execute('''
                INSERT INTO links_contactos (link_general, fecha_creacion, marca, descripcion)
                VALUES (?, ?, ?, ?)
            ''', (link_general.strip(), fecha_creacion.strftime("%Y-%m-%d"), marca.strip(), descripcion.strip()))
            conn.commit()
            st.success("Link Contactos creado exitosamente.")

# --- Página: Agregar Contactos ---
elif page == "Agregar Contactos":
    st.title("Agregar Contactos")
    st.markdown(disable_enter_js, unsafe_allow_html=True)
    
    df_links = pd.read_sql_query("SELECT * FROM links_contactos", conn)
    if df_links.empty:
        st.warning("No existen links. Por favor, crea un Link Contactos primero.")
    else:
        df_links['display'] = df_links.apply(
            lambda row: f"{row['link_general']} - {row['marca']}", axis=1
        )
        opcion = st.selectbox("Selecciona el Link Contactos", df_links['display'])
        selected_link = df_links[df_links['display'] == opcion].iloc[0]
        st.markdown(f"**Fecha de Creación:** {selected_link['fecha_creacion']}")
        st.markdown(f"**Marca:** {selected_link['marca']}")
        st.markdown(f"**Descripción del Link:** {selected_link['descripcion']}")
        
        link_id = selected_link["id"]
        
        with st.form("agregar_contacto_form"):
            link_auto = st.text_input("Link del Auto")
            telefono = st.text_input("Teléfono")
            nombre = st.text_input("Nombre")
            auto_modelo = st.text_input("Auto")
            precio_str = st.text_input("Precio (ej: 10,500,000)")
            descripcion_contacto = st.text_area("Descripción del Contacto")
            submitted_contacto = st.form_submit_button("Agregar Contacto")
        
        if submitted_contacto:
            if (not link_auto.strip() or not telefono.strip() or not nombre.strip() or 
                not auto_modelo.strip() or not precio_str.strip() or not descripcion_contacto.strip()):
                st.error("Todos los campos son requeridos.")
            else:
                try:
                    precio_clean = precio_str.replace(",", "").strip()
                    precio = float(precio_clean)
                except ValueError:
                    st.error("Precio inválido. Asegúrate de ingresar un número, por ejemplo: 10,500,000")
                    st.stop()
                try:
                    cursor = conn.cursor()
                    cursor.execute('''
                        INSERT INTO contactos (link_auto, telefono, nombre, auto, precio, descripcion, id_link)
                        VALUES (?, ?, ?, ?, ?, ?, ?)
                    ''', (link_auto.strip(), telefono.strip(), nombre.strip(), auto_modelo.strip(), precio, descripcion_contacto.strip(), link_id))
                    conn.commit()
                    st.success("Contacto agregado exitosamente.")
                except sqlite3.IntegrityError:
                    st.error("El teléfono ya existe. Por favor, ingresa un número diferente.")

# --- Página: Ver Contactos & Exportar ---
elif page == "Ver Contactos & Exportar":
    st.title("Ver Contactos & Exportar")
    df_links = pd.read_sql_query("SELECT * FROM links_contactos", conn)
    if df_links.empty:
        st.warning("No existen links. Por favor, crea un Link Contactos primero.")
    else:
        df_links['display'] = df_links.apply(
            lambda row: f"{row['link_general']} - {row['marca']}", axis=1
        )
        link_selected = st.selectbox("Selecciona el Link Contactos", df_links['display'])
        selected_link = df_links[df_links['display'] == link_selected].iloc[0]
        link_id = selected_link["id"]
        
        st.markdown(f"**Fecha de Creación:** {selected_link['fecha_creacion']}")
        st.markdown(f"**Marca:** {selected_link['marca']}")
        st.markdown(f"**Descripción del Link:** {selected_link['descripcion']}")
        
        st.subheader("Filtros de Búsqueda")
        filter_nombre = st.text_input("Filtrar por Nombre")
        filter_auto = st.text_input("Filtrar por Auto")
        filter_telefono = st.text_input("Filtrar por Teléfono")
        
        query = "SELECT * FROM contactos WHERE id_link = ?"
        params = [link_id]
        if filter_nombre:
            query += " AND nombre LIKE ?"
            params.append(f"%{filter_nombre}%")
        if filter_auto:
            query += " AND auto LIKE ?"
            params.append(f"%{filter_auto}%")
        if filter_telefono:
            query += " AND telefono LIKE ?"
            params.append(f"%{filter_telefono}%")
        
        df_contactos = pd.read_sql_query(query, conn, params=params)
        
        st.subheader("Contactos Registrados")
        st.dataframe(df_contactos)
        
        if not df_contactos.empty:
            output = BytesIO()
            with pd.ExcelWriter(output, engine='xlsxwriter') as writer:
                df_contactos.to_excel(writer, index=False, sheet_name='Contactos')
            processed_data = output.getvalue()
            st.download_button(
                label="Descargar Excel",
                data=processed_data,
                file_name="contactos.xlsx",
                mime="application/vnd.openxmlformats-officedocument.spreadsheetml.sheet"
            )
