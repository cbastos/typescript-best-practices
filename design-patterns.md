Aunque lo sepamos, creo que lo correcto es que empecemos por el principio: *¿qué es un patrón de diseño?* La definición según [Wikipedia](https://es.wikipedia.org/wiki/Patr%C3%B3n_de_dise%C3%B1o) es:

> Los patrones de diseño son unas **técnicas para resolver problemas comunes en el desarrollo de software** y otros ámbitos referentes al diseño de interacción o interfaces.
> Un patrón de diseño resulta ser una **solución a un problema de diseño**. Para que una solución sea considerada un patrón debe poseer ciertas características. Una de ellas es que **debe haber comprobado su efectividad** resolviendo problemas similares en ocasiones anteriores. Otra es que **debe ser reutilizable**, lo que significa que es aplicable a diferentes problemas de diseño en distintas circunstancias.

Y es que, efectivamente, un patrón de diseño no es más que una especie de receta o recomendaciones que podemos aplicar sobre nuestro código para resolver problemas de diseño comunes, incrementando además la calidad de éste, haciéndolo más mantenible, desacoplado y tolerante a cambios. Estas recetas podemos reutilizarlas gracias a que ya han sido probadas anteriormente en situaciones similares por gente que sabe mucho más que nosotros, por lo que su efectividad está más que demostrada.

Los patrones de diseño son uno de los aspectos más importantes de las **bases del desarrollo de software**, que siempre han estado (y estarán) ahí, independientemente de los frameworks y librerías que han ido saliendo al mercado, por lo que es casi obligatorio conocerlos.

Pero cuidado, no dejan de ser una herramienta más que tenemos disponible para desarrollar, por lo que deberemos conocer correctamente la motivación de cada uno de ellos, para saber cuando merece la pena aplicarlos sobre nuestros diseños, **evitando la sobreingeniería**.

A lo largo del artículo, veremos varios de estos **patrones aplicados sobre ejemplos reales** dentro de una aplicación *Angular* en la que trabajamos durante 2 años, implementada con *TypeScript*. De esta forma, evitaremos repasar cada patrón sólo desde el punto de vista teórico, por lo que os resultará más fácil saber si se ajusta a vuestras necesidades o no.

Antes de meternos de lleno con cada patrón, cabe destacar que se suelen agrupar en distintas categorías:
* **Patrones creacionales:** Solucionan problemas de creación de instancias de objetos.
* **Patrones estructurales:** Solucionan problemas de composición y agregación de objetos.
* **Patrones de comportamiento:** Solucionan problemas surgidos sobre las responsabilidades e interacciones entre objetos.
* **Patrones de UI:** Solucionan problemas específicos sobre la capa de presentación de la aplicación.

# Patrones creacionales
## Factory method
Nuestro objetivo antes de aplicar este patrón, es crear instancias de un objeto cualquiera *ConcreteProduct*, que puede implementar una interfaz *Product* en caso de que necesitemos instanciar distintos tipos de objetos con una estructura similar:
```ts
interface Product {
  // Contract members
}

class ConcreteProduct1 implements Product {
  // Contract members implemented
}
```
El patrón *factory method* nos permite **encapsular** la lógica de creación de ese objeto dentro de las subclases *ConcreteCreators*, descargando de dependencias al cliente que las consume:
```ts
abstract class Creator {
  public abstract factoryMethod(): Product;
}

class ConcreteCreator extends Creator {
  factoryMethod(): Product {
    return new ConcreteProduct1();
  }
}

export class Client {
  createProduct(): Product {
    const creator: Creator = new ConcreteCreator();
    return creator.factoryMethod();
  }
}
```
Como podéis ver en el código, *ConcreteCreator* extiende la clase abstracta *Creator* para que podamos asegurarnos de que todas las subclases encargadas de crear las instancias de los distintos tipos de objetos, devuelven un objeto de tipo *Product* y además tienen su correspondiente *factory method* con la misma firma. Además, podemos añadir en esa clase abstracta, más métodos que tengan que ver con la manipulación de los productos.

Además, este patrón favorece la **extensibilidad** siguiendo el [principio de open-close](https://es.wikipedia.org/wiki/Principio_de_abierto/cerrado): podemos añadir un nuevo *ConcreteProduct* con solo cumplir la interfaz, sin tener que modificar los productos existentes. Incluso tenemos la **versatilidad** de intercambiar facilmente un *ConcreteCreator* por otro.

Otra ventaja de este patrón, es que si los objetos *Product* tienen alguna dependencia externa, podemos proporcionárselas mediante su constructor y esta lógica estará encapsulada también. De tal forma que si en el futuro varía alguna de estas dependencias, el punto de colisión en el código es mucho menor, sólo tendremos que modificar la dependencia en la instanciación desde su *ConcreteProduct*, en vez de tener que revisar todos los *Client*.

Una vez recordada la teoría del patrón, vamos a ver cómo lo aplicamos nosotros en el proyecto.

Para el entorno de *Producción*, nos surgió la necesidad de guardar una traza de los eventos y errores que iban sucediendo durante la ejecución de la aplicación en una plataforma *cloud* como es [Application Insights](https://docs.microsoft.com/es-es/azure/azure-monitor/app/app-insights-overview). Sin embargo, para el entorno de *Desarrollo*, queríamos que esos eventos y errores aparecieran en la consola del navegador.

Para ello, definimos la interfaz *Logger* con el método *log* que usaremos para registrar estos eventos y errores. Dicha interfaz la implementan *AppInsightsLogger* que registra la información en la nube y un hipotético *ConsoleLogger* que no hace falta implementarlo puesto que el objeto *console* del navegador ya cumple con esa interfaz. 
```ts
interface Logger {
    log(message: string): void;
}

class AppInsightsLogger implements Logger {
    log(message: string): void {
        (<any>window).appInsights.trackTrace({ message });
    }
}
```
Para crear cada uno de los *loggers* tenemos dos clases *AppInsightsLoggerCreator* y *ConsoleLoggerCreator* que pueden ser instanciadas desde cualquier punto de la aplicación, **sin necesidad de conocer cómo deben crearse dichos objetos**, únicamente invocando al *factory method* *getLogger*:
```ts
class ConsoleLoggerCreator
    extends AbstractLoggerCreator {

    protected createLogger(): Logger {
        return console;
    }
}

class AppInsightsLoggerCreator
    extends AbstractLoggerCreator {

    protected createLogger(): Logger {
        return new AppInsightsLogger();
    }
}
```
Nótese que ambos *LoggerCreators* extienden la clase abstracta *AbstractLoggerCreator* para evitar que ciertos puntos de la aplicación dependan de un tipo concreto, favoreciendo implementar un **código más genérico y sin duplicidades**. Además, nos permite añadir en dicha clase más métodos que tengan que ver con la manipulación de loggers.
```ts
abstract class AbstractLoggerCreator {
    protected abstract createLogger(): Logger;

    public getLogger(): Logger {
        return this.createLogger();
    }
    // More possible methods about loggers manipulation
}
```
En vez de invocar directamente al *factory method* desde el cliente, implementamos una clase intermedia *LoggerCreator* que era la encargada de llamar a este método dependiendo del entorno en el que nos encontrábamos:
```ts
class LoggerCreator {
    public static getLogger(environment): Logger {
        const dictionary = {
            ['DEV']: () => new ConsoleLoggerCreator(),
            ['PROD']: () => new AppInsightsLoggerCreator()
        };
        return dictionary[environment]().getLogger();
    }
}

export class Client {
    constructor(private appConfig: { environment: string }) { }

    main() {
        LoggerCreator.getLogger(this.appConfig.environment).log('a message to log');
    }
}
```

## Factory method (with params)
Partimos del objetivo de instanciar tres tipos de objetos diferentes pero con una interfaz común a implementar:
```ts
interface Product {
  // Contract members
}

class ConcreteProduct1 implements Product {
  // Contract members implemented
}

class ConcreteProduct2 implements Product {
  // Contract members implemented
}

class ConcreteProduct3 implements Product {
  // Contract members implemented
}
```
Este patrón no deja de ser una variación sobre el patrón *factory method* original, cuya principal diferencia es que **un mismo factory method** de un *ConcreteCreator* puede ser **utilizado para crear estos tres tipos de objetos diferentes**, sólamente pasando por parámetro distintos valores.
```ts
interface Creator {
  factoryMethod(param: 1 | 2 | 3): Product;
}

class ConcreteCreator implements Creator {
  factoryMethod(param: 1 | 2 | 3): Product {
    const dictionary = {
      1: ConcreteProduct1,
      2: ConcreteProduct2,
      3: ConcreteProduct3
    }
    const ConcreteProduct = dictionary[param];
    return new ConcreteProduct();
  }
}

export class Client {
  createProduct(): Product {
    const creator = new ConcreteCreator();
    return creator.factoryMethod(1);
  }
}
```
En nuestro caso, teníamos una aplicación basada en formularios, donde los usuarios de cada equipo introducían la información necesaria. Esa información debía ser corregida y validada campo a campo, formulario a formulario por un jefe de equipo. 

Esta funcionalidad de modificación y validación del valor de cada campo se centralizó en un único componente, puesto que lo único que cambiaba era el tipo de datos que recibía y su forma de representación. De esta forma, tenemos:
```ts
@Component({ selector: 'fm-creator-component', template: '' })
export class FactoryMethodCreatorComponent implements OnInit {
    @Input() mode: ComponentModes;
    @Input() data: Array<any>;

    constructor(
        private componentFactoryResolver: ComponentFactoryResolver,
        public viewContainerRef: ViewContainerRef
    ) { }

    ngOnInit() {
        const componentToInstance = ComponentFactory.getComponentByMode(this.mode);
        const componentFactory =
            this.componentFactoryResolver.resolveComponentFactory(componentToInstance);

        this.viewContainerRef.clear();

        const componentInstance =
            this.viewContainerRef.createComponent(componentFactory).instance;
        componentInstance.data = this.data;
    }
}
```
Un componente *FactoryMethodCreatorComponent* que es el que se encarga de actuar como *creator* y que no tiene una representación propia, sino que es un mero contenedor que dentro de él renderiza el componente concreto en base al parámetro *mode* que le llega por parámetro como *Input*. Como véis, en nuestro caso no hay un *factory method* que se llame de forma explícita, ya que la "magia" transcurre dentro del método *ngOnInit*.

Por tanto, con esto ya éramos capaces de renderizar en una vista, de forma distinta, una serie de datos con el **componente *creator* y el parámetro como únicas dependencias**. Como se ve en el ejemplo, podemos representar los mismos datos con una caja de texto, un *dropdown* o un *textarea*, usando el mismo componente *creator* y pasándole valores distintos de *mode*.
```ts
@Component({ selector: 'fm-textarea-component', templateUrl: './textarea.component.html' })
export class FactoryMethodTextAreaComponent implements IComponent {
    @Input() data: Array<any>;
}
@Component({ selector: 'fm-text-component', templateUrl: './text.component.html' })
export class FactoryMethodTextComponent implements IComponent {
    @Input() data: Array<any>;
}
@Component({ selector: 'fm-select-component', templateUrl: './select.component.html' })
export class FactoryMethodSelectComponent implements IComponent {
    @Input() data: Array<any>;
}
```
Adicionalmente, definimos una interfaz *IComponent* para obligar a que los componentes ligados a las distintas representaciones tengan al menos la misma propiedad *Input* donde recibir los datos a representar. 

Esta relación entre el modo y el tipo de componente a renderizar, la definimos en un diccionario que posteriormente encapsulamos en una factoría que es consultada por el componente *creator*: 
```ts
export class ComponentFactory {
    static getComponentByMode(mode: ComponentModes) {
        const dictionary = {
            [ComponentModes.Text]: FactoryMethodTextComponent,
            [ComponentModes.TextArea]: FactoryMethodTextAreaComponent,
            [ComponentModes.Select]: FactoryMethodSelectComponent
        };
        return dictionary[mode];
    }
}
```

## Singleton
Este es sin duda uno de los patrones que más hemos utilizado todos en nuestros proyectos. Su característica principal es que nos permite **crear una instancia única** de un objeto determinado, consiguiendo así un punto centralizado al que podemos acceder desde cualquier punto de nuestro sistema.

Así, desde el punto de vista teórico, tendremos nuestra clase *Singleton* que creará y devolverá mediante el método *build* una instancia de sí misma la primera vez que se llame y reutilizará y devolverá esa misma instancia en el resto de llamadas sucesivas:
```ts
class Singleton {
    private static instance: Singleton;
    private constructor() { }
    static build(): Singleton {
        Singleton.instance = Singleton.instance || new Singleton();
        return Singleton.instance;
    }
}

export class Client {
    getInstance(): Singleton {
        const instance = Singleton.build();
        return instance;
    }
}
```
Con esto conseguimos también **evitar la creación de intancias innecesarias**. Hay que tener especial cuidado a la hora de acceder a esa instancia *Singleton* desde distintos puntos, puesto que si lo hacemos sin un orden establecido, obtendremos valores y resultados no esperados.

En nuestro proyecto real, teníamos una parte de generación de informes, donde se utilizaban una serie de templates HTML en los que se representaban una serie de datos almacenados en *data sources*. Todo esto gracias a la librería [Stimulsoft JS](https://www.stimulsoft.com/en/get-started/reports-js). 

Para obtener esos templates HTML, teníamos que hacer llamadas a una API externa. Para ello, creamos un agente intermedio que era el que realizaba las llamadas HTTP necesarias:
```ts
class ReportTemplatesAgent {
    constructor(
        private baseUrl: string,
        private applicationId: string
    ) { }

    getReporTemplates(): Promise<Array<ReportTemplate>> {
        return fetch(
            new Request(`${this.baseUrl}/get-all-report-templates`)
        ).then(response => response.json());
    }
}
```
El problema, es que antes de ejecutar esas llamadas HTTP, necesitábamos configurar la URL de la API al iniciar la aplicación SPA. Para ello, implementamos una factoría *ReportTemplatesAgentFactory* que es la que crea el objeto *ReportTemplatesAgent*. Como el inicio de nuestra aplicación y la obtención de las templates HTML se realizan en momentos distintos, esta instancia se crea en forma de *Singleton*:
```ts
class ReportTemplatesAgentFactory {
    private static instance: ReportTemplatesAgent;
    private static config: { baseUrl: string, applicationId: string };

    static use(baseUrl: string, applicationId: string) {
        this.config = { baseUrl, applicationId };
    }

    static build(): ReportTemplatesAgent {
        if (!this.config) {
            throw new Error('You should configure the factory before using it.')
        }
        this.instance = this.instance || new ReportTemplatesAgent(
            this.config.baseUrl,
            this.config.applicationId
        );
        return this.instance;
    }
}

export class Client {
    initializeApp(): void {
        ReportTemplatesAgentFactory.use(
            'https://my-base-url', 'my-app-id');
    }

    async generateReports(): Promise<void> {
        const reportTemplates =
            await ReportTemplatesAgentFactory
                .build().getReporTemplates();
        // Apply datasource over the report templates.
    }
}
```

# Patrones estructurales
## Adapter
El patrón *Adapter* o adaptador, nos sirve exactamente para lo que su propio nombre indica: para adaptar **una interfaz** específica, haciéndola compatible con otra, cuando a priori no lo son. Lo de **una interfaz** lo resalto de nuevo para cuando más adelante veamos el patrón *Facade*, puesto que es una de las principales diferencias entre ellos. 

Desde el punto de vista teórico, partimos de la interfaz *Adaptee* que queremos adaptar, y de la interfaz *Target* a la que queremos adaptarla:
```ts
class Adaptee {
    oldMethod(): void { }
}
interface Target {
    requiredMethod(): void;
}
```
Finalmente, definimos la clase *Adapter* que implementa la interfaz *Target* y recibirá en su constructor un objeto de la interfaz a adaptar (*Adaptee*). De esta forma, el método de la nueva interfaz que implementa, llamará por debajo al método no adaptado y lograremos nuestro objetivo:
```ts
class Adapter implements Target {
    constructor(private adapteeObject: Adaptee) { }

    requiredMethod(): void {
        this.adapteeObject.oldMethod();
    }
}
export class Client {
    static main(): void {
        const a_compatible_object = new CompatibleObject();
        const a_non_compatible_object = new Adaptee();
        const adapter = new Adapter(a_non_compatible_object);

        a_compatible_object.requiredMethod();
        adapter.requiredMethod();
    }
}
```
En nuestro caso, utilizamos este patrón para encapsular ciertos componentes que sabíamos que iban a ser muy cambiantes debido a la indefinición inicial que teníamos del producto. Con ello, **protegeríamos al resto del sistema de dichos cambios**.

Por ejemplo, implementamos un componente personalizado *tooltip* que adaptaba los eventos y propiedades de la librería de terceros *ng2-tooltip-directive*. En este ejemplo, podemos ver como adaptamos el evento *shown* disparado por la directiva de *Angular* *tooltip* a un *Output* personalizado:
```html
<i [tooltip]="text" (events)="handleTooltipEvents($event)" class="material-icons">info</i>
```
```ts
@Component({
    selector: 'fa-tooltip',
    templateUrl: './tooltip-adapter.component.html'
})
export class TooltipAdapterComponent {
    @Input() text: string;
    @Output() shown: EventEmitter<void> = new EventEmitter<void>();

    handleTooltipEvents(event: { type: string, position: DOMRect }) {
        if (event.type === 'shown') {
            this.shown.emit();
        }
    }
}
```
Si examinamos cómo debemos consumir dicho componente desde las vistas de la aplicación:
```html
<div class="container">
    <span>One text</span>
    <fa-tooltip class="tooltip" text="one tooltip" (shown)="tooltipWasShown()"></fa-tooltip>
    <span>Another text</span>
    <fa-tooltip class="tooltip" text="another tooltip" (shown)="tooltipWasShown()"></fa-tooltip>
</div>
```
Podemos observar que cualquier cambio en la librería de terceros, **sólo afectará** a nuestro componente personalizado, puesto que la dependencia está centralizada en él. 

Este caso específico de la encapsulación del *tooltip* resultó ser una buenísima decisión, puesto que pasados unos meses después del inicio del proyecto, cambiaron los requisitos de negocio respecto al *tooltip*, y no tuvimos más que cambiar la librería por otra y adaptar nuestro componente, el resto del sistema permaneció inalterado.

## Facade
Este patrón es fácilmente confundible con el patrón *Adapter* anterior, de hecho, nosotros mismos los hemos confundido a la hora de identificarlos, ¡no tengáis miedo a equivocaros!. A continuación, voy a intentar aclarar la diferencia entre ambos.

Nuestra principal motivación para este patrón es crear una interfaz (que actuará como *fachada*) para **simplificar la interacción y comunicación con otros subsistemas**. Imaginemos que tenemos tres subsistemas con tres métodos distintos:
```ts
class Part1 {
    method1(): void {
        console.log("method 1 (Part 1)");
    }
}

class Part2 {
    method2(): void {
        console.log("method 2 (Part 2)");
    }
}

class Part3 {
    method3(): void {
        console.log("method 3 (Part 3)");
    }
}
```
Estos métodos de cada subsistema, deben ser llamados siempre en un orden concreto. Podemos simplificar las llamadas ordenadas de cada método y centralizarlas en una clase *Facade*:
```ts
class Facade {
    private part1: Part1 = new Part1();
    private part2: Part2 = new Part2();
    private part3: Part3 = new Part3();

    operation1(): void {
        this.part1.method1();
        this.part2.method2();
    }

    operation2(): void {
        this.part1.method1();
        this.part3.method3();
    }
}
```
Con ello, además de simplificar las llamadas a los métodos, logramos **encapsular los subsistemas** y los clientes o consumidores no dependerán de ellos. Por tanto, podríamos cambiar la implementación interna de cada subsistema o incluso cambiar la *fachada* siempre que se cumpla la misma interfaz:
```ts
export class Client {
    callOperations(): void {
        const facade = new Facade();
        facade.operation1();
        facade.operation2();
    }
}
```
Y aquí podemos ver la **principal diferencia** respecto al patrón *Adapter*. El patrón *Adapter* adapta una interfaz para hacerla compatible con otra, el *Facade* crea y expone una interfaz simplificada para comunicarse con otras interfaces o subsistemas más complejos.

En nuestro proyecto, volviendo al ejemplo de los informes, para obtener sus templates HTML, debíamos utilizar diferentes tipos de servicios. Nuestra aplicación tenía dos posibles modos de ejecución: como una SPA convencional en un navegador web y como una aplicación de escritorio distribuida con [Electron](https://www.electronjs.org/).

En el caso en que estemos ejecutando la aplicación con *Electron*, los templates HTML debíamos obtenerlos desde los *assets* embebidos en la propia instalación de la aplicación. Por tanto, para este proceso, teníamos tres servicios implicados: un servicio para obtener el HTML de los *assets* (*Electron*), otro para obtenerlo en la versión web desde la API y otro para comprobar si estamos en el navegador web o en el escritorio.

Estos servicios podemos verlos a continuación:
```ts
export class AppModeService {
    isOffline(): boolean {
        return !!window.navigator.userAgent.match(/Electron/);
    }
}
export class AssetsService {
    getJsonFromFile(fileName: string): Promise<any> {
        return fetch(new Request(`./my-assets-path/${fileName}.json`))
            .then(response => response.json());
    }
}
export class ReportsService {
    async getAll(): Promise<IReport[]> {
        const reports = await fetch(
            new Request('http://my-reports-rest-api/reports'))
            .then(response => response.json());

        return reports.map((report, index) =>
            ({
                id: index,
                template: 'a-html-report-template',
                name: report.name
            }));
    }
}
```
Toda esta lógica para obtener los templates, la simplificamos creando un *ReportsAgentFacade* que actuaba como *fachada*, exponiendo únicamente el método *loadReports* para cargar los informes al inicio de la aplicación y otro método *getReport* para obtener un informe específico y generarlo en otro punto de la aplicación:
```ts
export class ReportsAgentFacade {
    private reportsService = new ReportsService();
    private assetsService = new AssetsService();
    private appModeService = new AppModeService();

    async loadReports(): Promise<IReport[]> {
        let reports: IReport[] = [];
        if (this.appModeService.isOffline()) {
            reports = await this.assetsService.getJsonFromFile('reports');
            localStorage.setItem('reports', JSON.stringify(reports));
        } else {
            reports = await this.reportsService.getAll();
        }
        return reports;
    }

    async getReport(reportId: number): Promise<IReport> {
        let reports: IReport[] = [];
        if (this.appModeService.isOffline()) {
            reports = JSON.parse(localStorage.getItem('reports'));
        } else {
            reports = await this.reportsService.getAll();
        }
        return reports.find(r => r.id === reportId);
    }
}
```

## Flyweight
El último de los patrones estructurales que veremos en este artículo, promueve solucionar problemas de rendimiento gracias a la **reducción del número de instancias** almacenadas en la memoria de un sistema. Se basa en, dada una lista de instancias de un tipo de objeto, agruparlas en base a sus propiedades intrínsecas (cuyos valores pueden ser repetidos entre varias instancias) para poder reutilizarlas, y que todas las propiedades extrínsecas (cuyos valores son específicos para cada instancia) sean proporcionadas al objeto en el momento en que se ejecutan sus métodos.

Para entenderlo mejor, vamos a ver un caso teórico/práctico: imaginemos que estamos trabajando con objetos de tipo coche y queremos optimizar nuestro sistema reduciendo el número de instancias que creamos de éstos. Para ello, primero debemos identificar las **propiedades intrínsecas**, las propiedades cuyos valores pueden ser repetidos en varias de las instancias del objeto. Para nuestro caso, estas propiedades podrían ser la marca y el color de cada coche, puesto que dos coches pueden coincidir en marca o color.

Después, identificaremos las **propiedades extrínsecas**, las propiedades cuyos valores son particulares de cada instancia. En nuestro caso, podría ser el kilometraje de cada vehículo. 

Una vez identificadas ambos tipos de propiedades, definimos una interfaz *Flyweight* que cumplirán todas las instancias que almacenemos de cada vehículo, y donde se definirán todas las operaciones que necesitemos que se ejecuten sobre ellos, pasando por parámetro las propiedades extrínsecas si fueran necesarias para dichas operaciones. Esta interfaz es implementada por cada instancia *ConcreteFlyweight* específica de cada coche:
```ts
interface Flyweight {
    operation(extrinsicState);
}

class ConcreteFlyweight implements Flyweight {
    operation(extrinsicState) { }
}
```
Estos objetos *ConcreteFlyweight* serán creados a partir de una factoría *FlyweightFactory*, que será la encargada de almacenar la instancia de cada *flyweight* en un diccionario interno indexado por las propiedades intrínsecas como clave. De este modo, la primera vez que recibamos unos valores específicos de las propiedades intrínsecas, se creará la instancia y se almacenará en el diccionario. La próxima vez que se reciban los mismos valores, se evitará la creación de una nueva instancia y se recuperará la que contenga el diccionario:
```ts
class FlyweightFactory {
    private flyweights: { [key: string]: Flyweight };

    getFlyWeight(intrinsicState): Flyweight {
        const key = this.getKey(intrinsicState);
        if (!this.flyweights[key]) {
            const newFlyweight = new ConcreteFlyweight();
            this.flyweights[key] = newFlyweight;
            return newFlyweight;
        } else {
            return this.flyweights[key];
        }
    }

    private getKey(intrinsicState): string {
        return `${intrinsicState.model}-${intrinsicState.color}`;
    }
}

export class Client {
    run() {
        const factory = new FlyweightFactory();
        const car1 = factory.getFlyWeight({ model: 'BMW', color: 'red' });
        const car2 = factory.getFlyWeight({ model: 'BMW', color: 'red' });
        const car3 = factory.getFlyWeight({ model: 'Toyota', color: 'white' });
        car1.operation({ km: 1000 });
        car2.operation({ km: 200 });
        car3.operation({ km: 3500 });
    }
}
```
En nuestro proyecto real, reinterpretamos este patrón para solucionar un problema de rendimiento importante con el que nos encontramos. Teníamos el típico componente modal para mostrar información al usuario. El problema es que este modal estaba referenciado (e instanciado por tanto) en cada componente que lo mostraba. Aplicando nuestro *flyweight* particular, identificamos que la propiedad intrínseca de nuestro objeto componente, era el propio modal, en todas las instancias eran iguales, con la excepción del texto que mostrábamos en cada momento (nuestra propiedad extrínseca).

Por tanto, definimos un componente *FlyweightDialogComponent*:
```ts
@Component({
    selector: 'fly-dialog',
    templateUrl: 'dialog.component.html',
    styleUrls: ['./dialog.component.css']
})
export class FlyweightDialogComponent {
    text = '';
    visible = false;
    show(text: string) { this.text = text; this.visible = true; }
    close() { this.visible = false; }
}
```
Que estará centralizado como **una instancia única** en un componente principal de la aplicación (en nuestro caso era el componente *app*):
```ts
@Component({
    selector: 'flyweight',
    templateUrl: './real-world.component.html',
    styleUrls: ['./real-world.component.css']
})
export class FlyweightRealWorldComponent {
    @ViewChild('dialog', { static: false }) dialog: FlyweightDialogComponent;
    showFlyweightDialog(extrinsicInfoText: string) {
        this.dialog.show(extrinsicInfoText);
    }
}
```
```html
<h3>Flyweight implementation:</h3>
<fly-refactored-info infoText="first info" (showDialog)=showFlyweightDialog($event)></fly-refactored-info>
<fly-refactored-info infoText="second info" (showDialog)=showFlyweightDialog($event)></fly-refactored-info>
<fly-dialog #dialog></fly-dialog>
```
Y cada vez que se referencie al diálogo modal, se ejecutará al método de *show* de la instancia única, proporcionándole el texto que debe mostrar en ese momento.

# Patrones de comportamiento
## Strategy
La motivación principal del patrón *Strategy* es tener la posibilidad de ejecutar distintas implementaciones para una misma funcionalidad de forma dinámica. De esta forma, imaginemos que tenemos que ejecutar una funcionalidad específica representada por un método *strategicMethod*:
```ts
interface Strategy {
    strategicMethod(): void;
}
```
Y dicha funcionalidad puede ser ejecutada de dos maneras (estrategias) diferentes:
```ts
class ConcreteStrategyA implements Strategy {
    public strategicMethod(): void {
        console.log('Strategy A');
    }
}

class ConcreteStrategyB implements Strategy {
    public strategicMethod(): void {
        console.log('Strategy B');
    }
}
```
Definimos una clase *Context* que conoce la implementación o estrategia que debe ejecutarse en cada momento para acometer la funcionalidad:
```ts
class Context {
    private strategy: Strategy;

    constructor(strategy: Strategy) {
        this.strategy = strategy;
    }

    setStrategy(strategy: Strategy) {
        this.strategy = strategy;
    }

    performStrategicMethod(): void {
        this.strategy.strategicMethod();
    }
}
```
Con ella, podemos desde el cliente **variar dinámicamente la estrategia a ejecutar**. Además, delegamos la implementación interna de cada comportamiento o estrategia a las subclases *ConcreteStrategy*. El punto negativo de este patrón, es que el cliente necesita conocer y referenciar las clases de cada una de las estrategias.
```ts
export class Client {
    run(): void {
        const context = new Context(new ConcreteStrategyA());
        context.performStrategicMethod();

        context.setStrategy(new ConcreteStrategyB());
        context.performStrategicMethod();
    }
}
```
En nuestro caso particular, registrábamos cada cambio en cada campo de cada formulario y lo almacenábamos en el estado centralizado de la aplicación. Utilizamos el patrón *Strategy* en el componente que tenía que representar este historial de cambios, ya que necesitábamos una representación diferente dependiendo del tipo de dato modificado.

Definimos por ello un formateador por cada posible tipo de dato que hubiera que representar:
```ts
export interface FieldFormatter<T> {
    format(field: Field<T>): string;
}
export class BooleanFormatter implements FieldFormatter<boolean> {

    format(field: Field<boolean>): string {
        return field.value ? 'True' : 'False';
    }
}
export class StringFormatter implements FieldFormatter<string> {

    format(field: Field<string>): string {
        return field.value;
    }
}
```
Y en el componente visual, utilizamos un servicio de *Angular*, que a través de una factoría, obtenía el formateador adecuado en base al tipo de dato de cada registro:
```ts
@Injectable()
export class FieldFormatterService {
    private formattersDictionary: { [key: number]: FieldFormatter<any> };

    constructor() {
        this.formattersDictionary =
            FieldTypeFormatsFactory.buildFormattersDictionary();
    }

    format(field: Field<any>): string {
        const typeFormatter = this.formattersDictionary[field.type];
        return typeFormatter !== undefined && typeFormatter.format(field);
    }
}
    
export class FieldTypeFormatsFactory {

    static buildFormattersDictionary(
    ): { [key: number]: FieldFormatter<any> } {
        return {
            [FieldTypes.String]: new StringFormatter(),
            [FieldTypes.Boolean]: new BooleanFormatter()
        };
    }
}

@Component({
    selector: 'strategy',
    templateUrl: './real-world.component.html'
})
export class StrategyRealWorldComponent implements OnInit {
    fieldsFormatted: Array<string>;

    constructor(private fieldFormatter: FieldFormatterService) { }

    ngOnInit() {
        const fields: Array<Field<any>> = [
            { type: FieldTypes.Boolean, value: true },
            { type: FieldTypes.String, value: 'a field value' }
        ]
        this.fieldsFormatted = fields.map(f => this.fieldFormatter.format(f));
    }
}
```
Como podéis ver en el ejemplo, casi siempre que aplicamos el patrón *Strategy*, se elige la estrategia o implementación a ejecutar en base a algún tipo de condición o parámetro. En nuestro caso, como hemos dicho, fue el tipo de dato.

## Observer
Este patrón seguramente es de los más utilizados después del *Singleton*. La motivación principal es **desacoplar la comunicación entre distintos objetos**. Desde el punto de vista teórico, empezamos definiendo los *Observers*:
```ts
interface Observer {
    update(): void;
}

class ConcreteObserver implements Observer {
    update(): void { }
}
```
Por otro lado, tendremos los objetos *Subjects*, que serán los encargados de tener una referencia directa a los *observers* para poder comunicarse con ellos:
```ts
interface Subject {
    registerObserver(observer: Observer);
    unregisterObserver(observer: Observer);
    notifyObservers();
}

class ConcreteSubject implements Subject {
    observers: Array<Observer>;

    registerObserver(observer: Observer) {
        this.observers.push(observer);
    }

    unregisterObserver(observer: Observer) {
        // Find and remove observer from the collection.
    }

    notifyObservers() {
        this.observers.forEach(o => o.update());
    }
}
```
Desde el cliente o cualquier punto del sistema, podremos crear los *observers* y suscribirlos a cada *subject* específico, de forma que podrán comunicarse con ellos y notificarles en cualquier momento mediante el método *notifyObservers*:
```ts
export class Client {
    run() {
        const subject = new ConcreteSubject();
        const observer1 = new ConcreteObserver();
        const observer2 = new ConcreteObserver();

        subject.registerObserver(observer1);
        subject.registerObserver(observer2);

        subject.notifyObservers();
    }
}
```
Como podéis observar, con este patrón logramos **desacoplar totalmente** a los *observers*, ya que no conocen ni tienen ninguna referencia a ningún *subject*. Además, no estamos atados a un orden concreto de notificación entre observers.

En nuestro caso particular, teníamos un estado centralizado implementado con *Redux* y debíamos notificar a cada uno de los componentes visuales cuando una propiedad del estado se veía modificada. Esta notificación la implementamos mediante el patrón *observer*, donde cada *observer* era cada uno de los componentes a los que se debía notificar:
```ts
@Component({
    selector: 'observer1',
    templateUrl: './observer1.component.html'
})
export class Observer1Component {
    public productNames: Array<string> = [];
    private model: Order;

    @Input() set order(orderFromState: Order) {
        this.model = orderFromState;
        this.buildFormattedOrderList();
    }
    get order() { return this.model; }

    constructor(private connector: ConnectorService) {
        this.connector.registerObserver(this);
    }

    private buildFormattedOrderList(): void {
        this.productNames = this.model.productList.map(p => p.name);
    }

    addNewProduct() {
        this.connector.updateState(
            addNewProductStateAction(this.model)
        );
    }
}
```
El *subject* o encargado de llevar a cabo dicha notificación, era un servicio de *Angular* al que bautizamos como *connector*:
```ts
@Injectable()
export class ConnectorService {
    private currentState: Order;
    private observers = [];

    constructor(private store: Store) {
        this.currentState = this.store.state;
    }

    registerObserver(component) {
        this.observers.push(component);
        this.notifyObserver(component);
    }

    updateState(newState: Order) {
        if (this.isStateChanged(newState)) {
            this.currentState = newState;
            this.notifyObservers();
        }
    }

    private notifyObservers() {
        this.observers.forEach(o => this.notifyObserver(o));
    }

    private notifyObserver(observer) {
        observer.order = this.currentState;
    }

    private isStateChanged(newState: Order): boolean {
        return this.currentState.id !== newState.id;
    }
}
```
Este servicio llevaba el registro de cada componente a notificar (que se autoregistraban en sus correspondientes constructores llamando al método *registerObserver*) y se suscribía él mismo a la *store* de *Redux* para saber cuando se modificaba cualquier propiedad y poder notificar a cada uno de los componentes.

# Patrones de UI
## Page-Object
Este último patrón que revisaremos en el artículo de hoy, no encajaba en ninguna de las categorías comunes, ya que su motivación es muy específica y está muy ligada a la parte de UI. La motivación es crear una abstracción de una vista en forma de objeto, de forma que **se desacoplan los consumidores de la propia vista**.

En nuestro caso, estos consumidores eran nuestros tests de componentes UI de *Angular*. Lo que hicimos es, en vez de acceder directamente al código HTML de cada componente desde el código de los tests, creamos un *PageObject* del componente:
```ts
export class ProductListPageObject {
    getProductId(): number {
        const productId = this.getHTMLElementByClass('id')[0].textContent;
        return parseInt(productId);
    }

    getProductName(): string {
        const productName = this.getHTMLElementByClass('name')[0].textContent;
        return productName;
    }

    openDetail(): void {
        const viewDetailButton = this.getHTMLElementByClass('view-detail')[0];
        viewDetailButton.click();
    }

    private getHTMLElementByClass(className: string): Array<HTMLElement> {
        return [new HTMLElement()];
    }
}
```
Que expone métodos *get* para obtener valores de la vista y algún método como *openDetail* para hacer *click* e interactuar sobre elementos de ésta:
```html
<ul>
    <li class="id">1</li>
    <li class="name">A product</li>
    <li><a href="#" class="view-detail">View Detail</a></li>
</ul>
<ul>
    <li class="id">2</li>
    <li class="name">Another product</li>
    <li><a href="#" class="view-detail">View Detail</a></li>
</ul>
```
Así, tendremos el código de nuestros tests totalmente desacoplado de la vista (además de ser mucho **más legible y mantenible**). En el caso en que cambie el HTML del componente (que suele ser muy habitual), sólo habrá que adaptar la clase *PageObject*, pero el código de nuestros tests quedará intacto, lo que conllevará que el mantenimiento de éstos sea mucho más fácil y no se acabarán abandonando.
```ts
export class TestClient {
    private pageObject = new ProductListPageObject();**

    is_a_valid_product_id(): boolean {
        const productId** = this.pageObject.getProductId();
        return productId > 0;
    }

    is_a_valid_product_name(): boolean {
        const productName = this.pageObject.getProductName();
        return productName.length < 255;
    }

    is_a_product_detail_visible(): boolean {
        this.pageObject.openDetail();
        // ...
        return false;
    }
}
```

# Y aún hay mas...
Estos han sido sólo algunos de los patrones que usamos en nuestra aplicación, si queréis conocer todos los que llegamos a aplicar, podéis consultar la [charla](https://www.youtube.com/watch?v=ZlhKj32KlfI) que impartimos en la JSDay Canarias 2019 y su [repositorio de código](https://github.com/cbastos/jsdaycan2019-typescript-patterns) correspondiente con ejemplos.

Además, no puedo dejar de recomendar el libro que más me ha servido de ayuda y que mejor explica la mayoría de patrones (para mi gusto): **Head First Design Patterns** de *Eric Freeman*, *Elisabeth Freeman*, *Kathy Sierra* y *Bert Bates*.

Espero que os haya resultado útil el artículo, y cualquier duda/pregunta/sugerencia podéis encontrarme en twitter como [@ivanirega](https://twitter.com/ivanirega). Y recordad: *"Donde hay patrón, no manda marinero"* XD