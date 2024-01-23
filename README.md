# SwiftUI Generic Service with Alamofire and URLSession
![Swift](https://img.shields.io/badge/Swift-5.9%20%7C%205.8%20%7C%205.7-orange.svg)
![Platform](https://img.shields.io/badge/Platform-iOS%20%7CMacOS-red.svg)
![License](https://img.shields.io/badge/License-MIT-blue.svg)

Bu örnek proje, SwiftUI kullanarak Dependency Injection (DI) ve Singleton tasarım desenini, MVVM ve Generic Network Manager oluşturmayı gösterir. 
Proje, Alamofire veya URLSession ile JSONPlaceholder API'den verileri çekmeyi amaçlar.

- SwiftUI
- Singleton
- Dependency Injection
- MVVM
- Generic Network Layer
- Protocol Oriented Programming (POP)
- Alamofire & URLSession

## ApiServiceProtocol Protocol

`ApiServiceProtocol` adlı bir protocol, ağ çağrıları yapmak için gerekli işlevselliği tanımlar.

```swift
protocol ApiServiceProtocol {
    func getRequest<T: Codable>(endpoint: URL, parameters: [String: Any]?, completion: @escaping(Result<T, Error>) -> Void )
    func addRequest<T: Codable>(endpoint: URL, data: T, completion: @escaping(Result<Void, Error>) -> Void)
    func updateRequest<T: Codable>(endpoint: URL, data: T, completion: @escaping(Result<Void, Error>) -> Void)
    func deleteRequest(endpoint: URL, completion: @escaping(Result<Void, Error>) -> Void)
}
```
## AlamofireApiService 
AlamofireApiService adlı sınıf, ApiService protocolünü uygular ve bu sınıfı singleton olarak tasarlar.

```swift
class AlamofireApiService: ApiServiceProtocol {
    
    private init() { }
    
    static let shared = AlamofireApiService()
    
    func getRequest<T: Codable>(endpoint: URL, parameters: [String: Any]?, completion: @escaping (Result<T, Error>) -> Void) {
        AF.request(endpoint, method: .get, parameters: parameters)
        .validate()
        .responseDecodable(of: T.self) { response in
            switch response.result {
            case .success(let value):
                completion(.success(value))
            case .failure(let error):
                completion(.failure(error))
            }
        }
    }

    // postRequest()
    // updateReques()
    // deleteRequest()
```

## ProductApiService
ProductApiService adlı sınıf, ağ işlemleri için DI ile kendisine tanımlanacak olan (ApiService protokülü uygulayan) api servis ile işlemlerini gerçekleştirir.

```swift
class ProductApiService{
    
    let apiService: ApiServiceProtocol
    
    init(apiService: ApiServiceProtocol) {
        self.apiService = apiService
    }
    
    func getAllProducts(parameters: [String: Any]?, completion: @escaping (Result<[Product], Error>) -> Void ){ // tüm postları getirmek için
        apiService.getRequest(endpoint: APIConstants.getProductEndpoint, parameters: parameters) { (result: Result<ProductResponse, Error>) in
            switch result {
            case .success(let data):
                completion(.success(data.products))
            case .failure(let error):
                completion(.failure(error))
            }
        }
    }
}
// getProductById()
// createProduct()
// updateProduct()
// deleteProduct()
```

## ProductViewModel
ProductViewModel, ürünleri çekmek ve ürün arayüzünü güncellemek için kullanılır. Dependency Injection kullanılarak ApiServiceProtocol enjekte edilir.

```swift
class ProductViewModel: ObservableObject {
    
    @Published var products: [Product] = []
    @Published var product: Product?
    @Published var isShowing: Bool = false
    @Published var isError: Bool = false
    @Published var title = ""
    @Published var message = ""
    
    let productApiService: ProductApiService
    
    init(productApiService: ProductApiService) {
        self.productApiService = productApiService
    }
    
    func fetchAllProducts(parameters: [String: Any]? = nil) {
        productApiService.getAllProducts(parameters: parameters) { result in
            switch result {
            case .success(let products):
                DispatchQueue.main.async {
                    self.products = products
                }
            case .failure(let error):
                print(error.localizedDescription)
            }
        }
    }
}

```
## View ile Kullanımı
Dependency Injection kullanılarak ApiServiceProtocol singleton olarak ProductViewModel'e enjekte edilir ve bu ViewModel, ProductListView tarafından kullanılır.

```swift
struct ProductListView: View {
    
    @StateObject var viewModel: ProductViewModel
    init(apiService:  ApiServiceProtocol) {
        _viewModel = StateObject(wrappedValue: ProductViewModel(productApiService: .init(apiService: apiService)))
    }

    var body: some View {
        // ...
    }
}

struct ContentView: View {
    var body: some View {
        NavigationStack {
            VStack {
                ProductListView(apiService: AlamofireApiService.shared)
            }
            .navigationTitle("Products")
        }
    }
}
```
