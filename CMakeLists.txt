cmake_minimum_required(VERSION 3.16)
project(MatchEngineMVP LANGUAGES CXX)

# C++20 obligatorio
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# Buscar automáticamente todos los archivos de código fuente y cabeceras
file(GLOB_RECURSE PROJECT_SOURCES 
    "${CMAKE_CURRENT_SOURCE_DIR}/*.cpp"
)
file(GLOB_RECURSE PROJECT_HEADERS 
    "${CMAKE_CURRENT_SOURCE_DIR}/*.h"
)

# Ejecutable con todos los .cpp encontrados (solo debería ser main.cpp)
add_executable(MatchEngine main.cpp)

# Incluir el directorio raíz para que los #include relativos funcionen desde cualquier lugar
target_include_directories(MatchEngine PRIVATE ${CMAKE_CURRENT_SOURCE_DIR})

# Añadir las cabeceras al proyecto para que aparezcan en el IDE (no se compilan, solo indexan)
target_sources(MatchEngine PRIVATE ${PROJECT_HEADERS})

# Opciones de compilación para GCC en Ubuntu (warnings útiles, optimización en Debug)
target_compile_options(MatchEngine PRIVATE -Wall -Wextra -Wpedantic)