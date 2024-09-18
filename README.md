# Documentação do Projeto AppDemoT4Face

## Arquitetura Geral

O projeto segue uma arquitetura **MVVM (Model-View-ViewModel)**, utilizando a tecnologia  **Kotlin** junto com framework **Jetpack Compose** para a construção da interface de usuário e o Retrofit para a comunicação com a API. Abaixo, estão descritas as principais camadas e seus componentes:

1. **Model**: Contém as classes que representam os dados manipulados no aplicativo, como `Face` e `PersonFace`. Esses modelos são utilizados para armazenar e transportar informações sobre as faces e suas características.
2. **Screens**: Define as telas do aplicativo, como `HomeScreen` e `AddFaceScreen`. Elas utilizam os componentes do Jetpack Compose para renderizar a interface de usuário, solicitar interações e navegar entre telas.
3. **ViewModel**: Responsável pela lógica de negócio e comunicação com a API. O `ApiViewModel` realiza operações como buscar as faces cadastradas e enviar novas faces para o backend.
4. **API**: A camada de API é implementada utilizando Retrofit, que facilita a comunicação com serviços REST. O `ApiService` define os endpoints disponíveis e como os dados serão enviados e recebidos da API.
5. **Utils**: Um utilitário para lidar com certificados SSL inseguros está presente na classe `UnsafeOkHttpClient`, o que permite a comunicação com servidores que utilizam certificados SSL auto-assinados ou não validados (uso recomendado apenas em ambiente de desenvolvimento).
6. **Components**: `AppBar` e `Header` componentes reutilizáveis 

---

### 1. API Service

O `ApiService.kt` define dois endpoints:

- `getPersonFaces`: realiza uma requisição GET para buscar a lista de faces cadastradas no servidor.
- `addPersonFace`: envia uma requisição POST com uma imagem e um nome para cadastrar uma nova face.

```kotlin
interface ApiService {
    @GET("/personface")
    suspend fun getPersonFaces(): List<PersonFace>

    @Multipart
    @POST("/personface/addface")
    suspend fun addPersonFace(
        @Part("name") name: RequestBody,
        @Part personFace: MultipartBody.Part
    ): retrofit2.Response<ResponseBody>
}
```
### 2. Model

Os modelos são simples e diretos, representando as entidades necessárias para a aplicação:
- `Face`: Representa uma face com um ID, nome e URL da imagem.
```kotlin
data class Face(val id: Int = 0, val name: String, val imageUrl: String?)
```
- `PersonFace`: Utilizada para armazenar informações mais detalhadas da face cadastrada.
```kotlin
data class PersonFace(val id: Int, val name: String, val feature: String)
```

### 3. Screens - Interface de Usuário

#### 3.1 AddFaceScreen

A tela `AddFaceScreen` permite que o usuário capture uma imagem usando a câmera e faça o cadastro dessa face no sistema. Utiliza o `CameraX` para captura de imagens
```kotlin
@Composable
fun AddFaceScreen(navController: NavController, apiViewModel: ApiViewModel = ApiViewModel()) {
    var name by remember { mutableStateOf("") }
    var capturedImage by remember { mutableStateOf<Bitmap?>(null) }
    
    Scaffold(topBar = { Header(navController = navController, title = "Cadastrar Face") }) {
        Column(modifier = Modifier.fillMaxSize().padding(paddingValues)) {
            TextField(value = name, onValueChange = { name = it }, label = { Text("Nome da Face") })

            if (capturedImage == null) {
                Button(onClick = { /* Ativar câmera */ }) {
                    Text("Ativar Câmera")
                }
            } else {
                Image(bitmap = capturedImage!!.asImageBitmap(), contentDescription = null)
                Button(onClick = { /* Cadastrar Face */ }) {
                    Text("Cadastrar Face")
                }
            }
        }
    }
}

```
#### 3.2 HomeScreen

A tela principal exibe a lista de faces cadastradas, utilizando um `LazyColumn` para carregar os dados de forma eficiente.

```kotlin
@Composable
fun HomeScreen(navController: NavController, apiViewModel: ApiViewModel = ApiViewModel()) {
    var faces by remember { mutableStateOf(listOf<PersonFace>()) }
    
    Scaffold(topBar = { Header(navController = navController, title = "Faces") }) {
        if (faces.isEmpty()) {
            Text("Nenhuma face disponível no momento")
        } else {
            LazyColumn {
                items(faces) { face ->
                    PersonFaceItem(personFace = face)
                }
            }
        }
    }
}

```

### 4. ApiViewModel

O ApiViewModel centraliza a lógica de comunicação com a API. Ele lida com operações como buscar e cadastrar faces, utilizando o Retrofit.

Principais Funções:
- `fetchPersonFaces:` Busca as faces cadastradas no servidor e atualiza a UI.
- `addPersonFace:` Envia uma imagem e nome para cadastrar uma nova face no backend.

```kotlin
class ApiViewModel : ViewModel() {
    private val retrofit = Retrofit.Builder()
        .baseUrl("https://137.184.100.1:9450")
        .client(getUnsafeOkHttpClient())
        .addConverterFactory(GsonConverterFactory.create())
        .build()

    private val apiService = retrofit.create(ApiService::class.java)

    fun fetchPersonFaces(onSuccess: (List<PersonFace>) -> Unit, onError: (Throwable) -> Unit) {
        viewModelScope.launch {
            try {
                val faces = apiService.getPersonFaces()
                onSuccess(faces)
            } catch (e: Exception) {
                onError(e)
            }
        }
    }

    fun addPersonFace(context: Context, name: String, image: Bitmap, onSuccess: () -> Unit, onError: (Throwable) -> Unit) {
        // Converte e envia o bitmap
    }
}

```
## 5. Funções de utilidade

### 5.1 getUnsafeOkHttpClient
Essa função cria um cliente `OkHttp` que desativa a verificação de certificados SSL. Isso pode ser útil para testes em ambientes de desenvolvimento com certificados auto-assinados, **mas não deve ser usado em produção**.

```kotlin
fun getUnsafeOkHttpClient(): OkHttpClient {
    val trustAllCerts = arrayOf<TrustManager>(object : X509TrustManager {
        override fun checkClientTrusted(chain: Array<X509Certificate>, authType: String) {}
        override fun checkServerTrusted(chain: Array<X509Certificate>, authType: String) {}
        override fun getAcceptedIssuers(): Array<X509Certificate> = arrayOf()
    })
    
    val sslContext = SSLContext.getInstance("SSL").apply {
        init(null, trustAllCerts, java.security.SecureRandom())
    }

    return OkHttpClient.Builder()
        .sslSocketFactory(sslContext.socketFactory, trustAllCerts[0] as X509TrustManager)
        .hostnameVerifier { _, _ -> true }
        .build()
}

```
### 5.2 convertBitmapToFile
Essa função converte um objeto `Bitmap` em um arquivo físico no sistema, o que é necessário para enviar a imagem da face via multipart para a API.

```kotlin
fun convertBitmapToFile(bitmap: Bitmap, file: File, quality: Int = 50): File {
    try {
        FileOutputStream(file).use { outputStream ->
            bitmap.compress(Bitmap.CompressFormat.JPEG, quality, outputStream)
        }
    } catch (e: IOException) {
        e.printStackTrace()
    }
    return file
}


```

### 5.3 createTempFile
Essa função cria um arquivo temporário no cache da aplicação para armazenar a imagem capturada da câmera antes de ser enviada à API.

```kotlin
fun createTempFile(context: Context): File {
    return File(context.cacheDir, "temp_image.jpg").apply {
        createNewFile()
    }
}


```

### 5.4 imageProxyToBitmap
Essa função converte o objeto `ImageProxy` (retornado pela câmera) em um objeto `Bitmap`, que pode ser exibido na interface do usuário e enviado ao backend.
```kotlin
private fun imageProxyToBitmap(image: ImageProxy): Bitmap {
    val buffer: ByteBuffer = image.planes[0].buffer
    val bytes = ByteArray(buffer.remaining())
    buffer.get(bytes)
    return BitmapFactory.decodeByteArray(bytes, 0, bytes.size)
}


```
## 6. Components

A camada de **components** contém componentes reutilizáveis que são usados nas telas do aplicativo para criar uma interface de usuário consistente. Esses componentes são projetados para serem independentes e reutilizáveis em diferentes partes da aplicação.

### 6.1 AppBar
O arquivo `AppBar.kt` define a barra de navegação superior, que inclui um título e um botão de menu. Esse componente é utilizado em várias telas do aplicativo para fornecer uma interface de navegação consistente.

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun AppBar(navController: NavController, title: String) {
    var expanded by remember { mutableStateOf(false) }

    TopAppBar(
        title = {
            Text(
                text = title,
                color = MaterialTheme.colorScheme.onPrimary
            )
        },
        actions = {
            Box {
                IconButton(onClick = {
                    expanded = !expanded
                }) {
                    Icon(
                        Icons.Outlined.Menu,
                        contentDescription = "Menu",
                        tint = Color.White
                    )
                }
                // Aqui o DropdownMenu é chamado
                DropdownMenu(
                    expanded = expanded,
                    onDismissRequest = {
                        expanded = false
                    },
                    navController = navController
                )
            }
        },
        colors = TopAppBarDefaults.topAppBarColors(
            containerColor = MaterialTheme.colorScheme.primary
        )
    )
}

```

### 6.2 DropdownMenu

O arquivo `DropdownMenu.kt` define o menu suspenso que é acionado na barra de navegação. Ele fornece opções de navegação adicionais, como "Faces" e "Cadastrar Face". Esse componente é exibido quando o botão de menu na AppBar é pressionado.
```kotlin
@Composable
fun DropdownMenu(
    navController: NavController,
    modifier: Modifier = Modifier,
    expanded: Boolean,
    onDismissRequest: () -> Unit,
) {
    androidx.compose.material3.DropdownMenu(
        expanded = expanded,
        onDismissRequest = onDismissRequest,
        content = {
            DropdownMenuItem(
                text = { Text(text = "Faces") },
                leadingIcon = {
                    Icon(
                        imageVector = Icons.Outlined.Home,
                        contentDescription = null,
                        tint = MaterialTheme.colorScheme.primary,
                    )
                },
                onClick = {
                    navController.navigate(AppScreen.Home.route)
                    onDismissRequest() // Fecha o menu
                }
            )
            DropdownMenuItem(
                text = { Text(text = "Cadastrar Face") },
                leadingIcon = {
                    Icon(
                        imageVector = Icons.Outlined.Face,
                        contentDescription = null,
                        tint = MaterialTheme.colorScheme.primary,
                    )
                },
                onClick = {
                    navController.navigate(AppScreen.AnotherScreen.route)
                    onDismissRequest() // Fecha o menu
                }
            )
        }
    )
}

```
## 6.3 Header

O arquivo `Header.kt` define um componente de cabeçalho reutilizável que inclui um título e um botão de menu. Ele é muito semelhante ao `AppBar`, mas foi separado como um componente reutilizável específico para ser usado em telas que requerem uma navegação mais simples.
```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun Header(navController: NavController, title: String) {
    var expanded by remember { mutableStateOf(false) }

    TopAppBar(
        title = {
            Text(
                text = title,
                color = MaterialTheme.colorScheme.onPrimary
            )
        },
        actions = {
            Box {
                IconButton(onClick = {
                    expanded = !expanded
                }) {
                    Icon(
                        Icons.Outlined.Menu,
                        contentDescription = "Menu",
                        tint = Color.White
                    )
                }
                DropdownMenu(
                    expanded = expanded,
                    onDismissRequest = { expanded = false },
                    navController = navController
                )
            }
        },
        colors = TopAppBarDefaults.topAppBarColors(
            containerColor = MaterialTheme.colorScheme.primary
        )
    )
}

```


### Considerações finais

A arquitetura MVVM utilizada separa bem a lógica de negócio da interface de usuário, facilitando a manutenção e evolução do código. O uso de componentes reutilizáveis, como `AppBar` e `Header`, também ajuda a manter a consistência visual e de navegação.
