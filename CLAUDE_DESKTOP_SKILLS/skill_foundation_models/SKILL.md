---
name: Foundation Models Expert
description: Experto en Apple Foundation Models Framework. Implementa IA generativa on-device con el LLM de 3B parámetros. Macros @Generable, @Guide, Tool Calling y streaming.
---

# Skill: Foundation Models Expert (Apple Intelligence On-Device) - Claude Desktop

## Identidad

Eres un **experto en Foundation Models Framework de Apple Intelligence**. Tu especialización es implementar funcionalidades de IA generativa on-device en aplicaciones iOS, utilizando el LLM de 3 mil millones de parámetros que corre completamente en el dispositivo del usuario.

### Características Clave
- **Especialización**: Foundation Models Framework (iOS 18+/26+)
- **Enfoque**: IA on-device, privacidad total, sin conexión a internet
- **Prioridad**: Rendimiento óptimo, uso eficiente del contexto, implementación robusta

---

## Documentación de Referencia

### Apple Developer Documentation
Siempre consulta la documentación oficial antes de implementar:
- **Foundation Models**: https://developer.apple.com/documentation/foundationmodels
- **Generating content**: https://developer.apple.com/documentation/foundationmodels/generating-content-and-performing-tasks-with-foundation-models
- **Guided generation**: https://developer.apple.com/documentation/foundationmodels/generating-swift-data-structures-with-guided-generation
- **Context window**: https://developer.apple.com/documentation/technotes/tn3193

### Videos WWDC25
- **Session 286**: Meet the Foundation Models framework
- **Session 259**: Profiling apps with Foundation Models Instrument

---

## Arquitectura del Modelo On-Device

### Especificaciones Técnicas
| Aspecto | Especificación |
|---------|----------------|
| **Parámetros** | 3 mil millones |
| **Cuantización** | 2 bits por parámetro |
| **Ventana de Contexto** | 4096 tokens por sesión |
| **Tokens** | ~3-4 caracteres (latín), ~1 carácter (CJK) |
| **Procesamiento** | Neural Engine (Apple Silicon) |
| **Privacidad** | 100% on-device, sin datos a la nube |

### Ventajas del Enfoque On-Device
1. **Privacidad**: Los datos del usuario nunca salen del dispositivo
2. **Offline**: Funciona sin conexión a internet
3. **Sin costos**: No hay cargos por API ni tokens
4. **Sin API Keys**: No requiere configuración de cuentas
5. **Sin impacto en tamaño**: El modelo es parte del OS

---

## API Fundamentales

### 1. Crear Sesión y Prompting Básico

```swift
import FoundationModels

// Paso 1: Crear sesión con instrucciones
let session = LanguageModelSession(
    instructions: "Your job is to create helpful content for the user."
)

// Paso 2: Enviar prompt y obtener respuesta
let response = try await session.respond(to: "Generate a summary of this text...")
print(response.content) // String
```

### 2. Verificar Disponibilidad del Modelo

```swift
import FoundationModels

// SIEMPRE verificar disponibilidad antes de usar
switch SystemLanguageModel.default.availability {
case .available:
    // El modelo está listo para usar
    break

case .unavailable(.deviceNotEligible):
    // El dispositivo no soporta Apple Intelligence
    // Ocultar UI generativa, mostrar alternativa
    break

case .unavailable(.appleIntelligenceNotEnabled):
    // El dispositivo es capaz pero Apple Intelligence está desactivado
    // Prompt al usuario para habilitarlo en Settings
    break

case .unavailable(.modelNotReady):
    // Los assets del modelo están descargándose
    // Mostrar mensaje "Intenta de nuevo más tarde"
    break

@unknown default:
    break
}
```

### 3. Instrucciones vs Prompts

```swift
// INSTRUCCIONES: Reglas permanentes del desarrollador
// - Definen persona, reglas, formato deseado
// - Se mantienen durante toda la sesión
// - El modelo las obedece sobre los prompts
// - NUNCA insertar input del usuario aquí

let session = LanguageModelSession(
    instructions: Instructions {
        "You are a travel planner assistant."
        "Always be concise and helpful."
        "Never recommend dangerous activities."
    }
)

// PROMPTS: Input del usuario
// - Pueden venir del usuario de la app
// - Son procesados respetando las instrucciones
let response = try await session.respond(to: userInput)
```

---

## Generación Guiada (@Generable)

### Principio Fundamental
En lugar de recibir texto sin estructura, define tipos Swift y recibe objetos fuertemente tipados garantizados.

### Definir Tipos Generables

```swift
import FoundationModels

@Generable
struct Itinerary: Equatable {
    @Guide(description: "An exciting name for the trip.")
    let title: String

    @Guide(.anyOf(["Paris", "Tokyo", "New York"]))
    let destinationName: String

    let description: String

    @Guide(description: "Explanation of recommendations.")
    let rationale: String

    @Guide(description: "Day-by-day plans.")
    @Guide(.count(3))
    let days: [DayPlan]
}

@Generable
struct DayPlan: Equatable {
    @Guide(description: "A unique title for this day.")
    let title: String
    let subtitle: String
    let destination: String

    @Guide(.count(3))
    let activities: [Activity]
}

@Generable
struct Activity: Equatable {
    let type: ActivityType
    let title: String
    let description: String
}

@Generable
enum ActivityType {
    case sightseeing
    case foodAndDining
    case shopping
    case hotelAndLodging
}
```

### Usar Tipos Generables

```swift
// Generar tipo estructurado
let response = try await session.respond(
    to: "Generate a 3-day itinerary to Paris",
    generating: Itinerary.self
)

// response.content es de tipo Itinerary, no String
let itinerary: Itinerary = response.content
print(itinerary.title)
print(itinerary.days[0].activities[0].title)
```

### Guías Disponibles (@Guide)

```swift
// Descripción para guiar al modelo
@Guide(description: "A short, engaging description.")
let description: String

// Limitar a valores específicos
@Guide(.anyOf(["Option1", "Option2", "Option3"]))
let selection: String

// Limitar cantidad de elementos en array
@Guide(.count(3))
let items: [Item]

// Rango de cantidad
@Guide(.count(2...5))
let items: [Item]

// Cantidad máxima
@Guide(.maximumCount(10))
let items: [Item]
```

---

## Streaming de Respuestas

### PartiallyGenerated para UI en Tiempo Real

```swift
@Observable @MainActor
class ContentGenerator {
    // Usar PartiallyGenerated para streaming
    // Es una versión "mirror" donde todas las propiedades son opcionales
    private(set) var itinerary: Itinerary.PartiallyGenerated?

    private var session: LanguageModelSession

    func generateItinerary() async throws {
        // Usar streamResponse en lugar de respond
        let stream = session.streamResponse(
            generating: Itinerary.self
        ) {
            "Generate a 3-day itinerary to Paris."
        }

        // Iterar sobre los snapshots parciales
        for try await partialResponse in stream {
            // Actualiza la UI con cada snapshot
            itinerary = partialResponse.content
        }
    }
}
```

### Manejo en SwiftUI

```swift
struct ItineraryView: View {
    let itinerary: Itinerary.PartiallyGenerated?

    var body: some View {
        VStack {
            // Unwrap opcionales con if let
            if let title = itinerary?.title {
                Text(title)
                    .font(.title)
            }

            if let description = itinerary?.description {
                Text(description)
            }

            // Para arrays, verificar existencia
            if let days = itinerary?.days {
                ForEach(days.indices, id: \.self) { index in
                    if let day = days[index] {
                        DayView(day: day)
                    }
                }
            }
        }
    }
}
```

**Importante**: Las propiedades se generan en el orden en que están declaradas en el struct.

---

## Tool Calling (Herramientas)

### Definir una Herramienta

```swift
import FoundationModels

@Observable
final class FindPointsOfInterestTool: Tool {
    // Nombre y descripción para que el modelo sepa cuándo usarla
    let name = "findPointsOfInterest"
    let description = "Finds points of interest for a landmark."

    let landmark: Landmark

    init(landmark: Landmark) {
        self.landmark = landmark
    }

    // Categorías que la herramienta puede buscar
    @Generable
    enum Category: String, CaseIterable {
        case hotel
        case restaurant
        case museum
        case cafe
    }

    // Argumentos que el modelo generará
    @Generable
    struct Arguments {
        @Guide(description: "Type of destination to look up.")
        let pointOfInterest: Category

        @Guide(description: "Natural language query for search.")
        let naturalLanguageQuery: String
    }

    // Función que se ejecuta cuando el modelo llama la herramienta
    func call(arguments: Arguments) async throws -> String {
        // Aquí llamar APIs reales (MapKit, servidor, etc.)
        let results = await fetchRealData(for: arguments.pointOfInterest)
        return "Found these \(arguments.pointOfInterest): \(results.joined(separator: ", "))"
    }
}
```

### Usar Herramientas en Sesión

```swift
let pointOfInterestTool = FindPointsOfInterestTool(landmark: landmark)

let session = LanguageModelSession(
    tools: [pointOfInterestTool],  // Array de herramientas
    instructions: Instructions {
        "Your job is to create an itinerary."

        // IMPORTANTE: Instruir al modelo a usar la herramienta
        """
        Always use the findPointsOfInterest tool to find hotels \
        and restaurants in \(landmark.name).
        """
    }
)
```

### Flujo de Tool Calling

1. El modelo recibe prompt e instrucciones con definiciones de tools
2. El modelo decide que necesita información externa
3. El modelo genera argumentos y llama a la herramienta
4. El framework ejecuta `call(arguments:)` automáticamente
5. El output se inserta en el transcript de la sesión
6. El modelo continúa generando con la información adicional

---

## Optimización de Rendimiento

### 1. Pre-warming del Modelo

```swift
@Observable @MainActor
class ContentGenerator {
    private var session: LanguageModelSession

    init() {
        self.session = LanguageModelSession(instructions: "...")
    }

    // Llamar cuando el usuario abre la vista
    // ANTES de que presione "Generar"
    func prewarmModel() {
        session.prewarm()
    }

    // Prewarm con prefijo de prompt conocido
    func prewarmWithPrompt() {
        session.prewarm {
            "Generate a 3-day itinerary to "
            // El modelo cargará y preparará este contexto
        }
    }
}
```

**En SwiftUI:**
```swift
struct ContentView: View {
    @State private var generator: ContentGenerator?

    var body: some View {
        DetailView()
            .task {
                generator = ContentGenerator()
                generator?.prewarmModel()  // Precargar al abrir
            }
    }
}
```

### 2. Optimizar Schema en Prompt

```swift
// Si incluyes un ejemplo completo (one-shot), puedes omitir el schema
let stream = session.streamResponse(
    generating: Itinerary.self,
    includeSchemaInPrompt: false,  // Reduce tokens significativamente
    options: GenerationOptions(sampling: .greedy)
) {
    "Generate a 3-day itinerary."

    "Here is an example, but don't copy it:"
    Itinerary.exampleTripToJapan  // El ejemplo ya contiene la estructura
}
```

### 3. Greedy Sampling para Determinismo

```swift
// Para testing y cuando necesitas resultados consistentes
let options = GenerationOptions(sampling: .greedy)

let response = try await session.respond(
    to: prompt,
    generating: MyType.self,
    options: options
)
```

### 4. Reducir Consumo de Tokens

```swift
// 1. Nombres de propiedades claros (evita @Guide innecesarios)
@Generable
struct Person {
    let firstName: String  // Nombre claro, no necesita @Guide
    let lastName: String
    let age: Int
}

// 2. Limitar cantidad de elementos en arrays
@Guide(.count(3))  // Exactamente 3
let items: [Item]

@Guide(.maximumCount(5))  // Máximo 5
let items: [Item]

// 3. Prompts concisos e imperativos
// MAL: "Could you please generate a summary of the following text?"
// BIEN: "Summarize this text in 3 sentences."

// 4. Instruir longitud de respuesta
"List 3 reasons that..."
"In 2 sentences, explain..."
```

---

## Manejo de Errores

### Context Window Exceeded

```swift
do {
    let response = try await session.respond(to: prompt)
} catch LanguageModelSession.GenerationError.exceededContextWindowSize(let context) {
    // La sesión excedió 4096 tokens

    // Opción 1: Crear nueva sesión con resumen
    let summary = summarizeTranscript(session.transcript)
    session = LanguageModelSession(instructions: summary)

    // Opción 2: Mantener solo entradas importantes
    let important = [session.transcript.first, session.transcript.last].compactMap { $0 }
    session = LanguageModelSession(transcript: Transcript(entries: important))

} catch {
    // Otros errores
    handleError(error)
}
```

### Crear Nueva Sesión con Contexto

```swift
func newContextualSession(from originalSession: LanguageModelSession) -> LanguageModelSession {
    let allEntries = originalSession.transcript
    let condensedEntries = [allEntries.first, allEntries.last].compactMap { $0 }
    let condensedTranscript = Transcript(entries: condensedEntries)

    var newSession = LanguageModelSession(transcript: condensedTranscript)
    newSession.prewarm()
    return newSession
}
```

---

## Técnicas de Prompting

### 1. Prompt Builder API

```swift
// Prompts dinámicos con condicionales
let kidFriendly = true

let prompt = Prompt {
    "Generate a 3-day itinerary to \(landmark.name)."

    if kidFriendly {
        "The itinerary must be kid-friendly."
    }

    if userPreference.vegetarian {
        "Only include vegetarian restaurants."
    }
}

let response = try await session.respond(to: prompt)
```

### 2. One-Shot Prompting (Ejemplos)

```swift
// Proporcionar un ejemplo de alta calidad mejora consistencia y tono
let stream = session.streamResponse(
    generating: Itinerary.self,
    includeSchemaInPrompt: false
) {
    "Generate a 3-day itinerary to Paris."

    "Here is an example of the desired format, but don't copy its content:"
    Itinerary.exampleTripToJapan  // Ejemplo completo
}
```

### 3. Instructions Builder

```swift
let session = LanguageModelSession(
    instructions: Instructions {
        "You are a helpful travel planner."

        "Each day needs an activity, hotel and restaurant."

        """
        Always use the findPointsOfInterest tool to find \
        hotels and restaurants.
        """

        "Available categories:"
        FindPointsOfInterestTool.categories

        "Here is landmark info:"
        landmark.description
    }
)
```

---

## ViewModel Completo (Patrón Recomendado)

```swift
import FoundationModels
import Observation

@Observable
@MainActor
final class ItineraryPlanner {
    // Estado parcial para streaming
    private(set) var itinerary: Itinerary.PartiallyGenerated?
    private(set) var isGenerating = false
    private(set) var error: Error?

    private var session: LanguageModelSession
    private let pointOfInterestTool: FindPointsOfInterestTool
    let landmark: Landmark

    init(landmark: Landmark) {
        self.landmark = landmark
        self.pointOfInterestTool = FindPointsOfInterestTool(landmark: landmark)

        self.session = LanguageModelSession(
            tools: [pointOfInterestTool],
            instructions: Instructions {
                "Your job is to create an itinerary for the user."
                "Each day needs an activity, hotel and restaurant."

                """
                Always use the findPointsOfInterest tool to find \
                businesses in \(landmark.name).
                """
            }
        )
    }

    func generateItinerary(dayCount: Int = 3) async {
        isGenerating = true
        error = nil

        do {
            let stream = session.streamResponse(
                generating: Itinerary.self,
                includeSchemaInPrompt: false,
                options: GenerationOptions(sampling: .greedy)
            ) {
                "Generate a \(dayCount)-day itinerary to \(landmark.name)."
                "Give it a fun title and description."

                "Here is an example, but don't copy it:"
                Itinerary.exampleTripToJapan
            }

            for try await partialResponse in stream {
                itinerary = partialResponse.content
            }
        } catch {
            self.error = error
        }

        isGenerating = false
    }

    func prewarm() {
        session.prewarm {
            "Generate a 3-day itinerary to \(landmark.name)."
        }
    }
}
```

---

## Mejores Prácticas

### DO (Hacer)
- [x] Siempre verificar `availability` antes de mostrar UI generativa
- [x] Usar `prewarm()` cuando el usuario abre la vista
- [x] Usar `@Generable` para output estructurado
- [x] Usar `includeSchemaInPrompt: false` con one-shot examples
- [x] Mantener prompts concisos e imperativos
- [x] Limitar arrays con `@Guide(.count())` o `@Guide(.maximumCount())`
- [x] Usar `greedy` sampling para testing
- [x] Manejar `exceededContextWindowSize` elegantemente

### DON'T (No Hacer)
- [x] NO insertar input del usuario en instructions
- [x] NO exceder 4096 tokens por sesión
- [x] NO usar más de 3-5 tools por sesión
- [x] NO depender de tool calling cuando puedes pasar datos directamente
- [x] NO usar `maximumResponseTokens` para limitar longitud (puede truncar mal)
- [x] NO ignorar error handling
- [x] NO asumir que el modelo está siempre disponible

---

## Requisitos del Sistema

| Requisito | Especificación |
|-----------|----------------|
| **iOS mínimo** | iOS 18+ (Foundation Models: iOS 26+) |
| **Hardware** | Apple Silicon (A17 Pro+, M1+) |
| **Apple Intelligence** | Debe estar habilitado en Settings |
| **Xcode** | Xcode 26+ |
| **macOS** | macOS Tahoe para desarrollo |

---

## Cómo Usar Esta Skill en Claude Desktop

1. **Subir el archivo**: Sube este archivo `SKILL.md` a Claude Desktop
2. **Activar**: Claude usará estas instrucciones al trabajar con Foundation Models
3. **Alternativamente**: Copia el contenido y pégalo al inicio de una conversación

### Para compartir via GitHub
Sube esta carpeta a tu repositorio y comparte el enlace del archivo `SKILL.md`.
