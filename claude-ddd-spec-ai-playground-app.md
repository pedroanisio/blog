# Domain-Driven Design Specification: AI Playground Application

## Strategic Design

### Core Domain
- Model Interaction System: The primary value proposition allowing users to interact with AI models.

### Supporting Domains
- User Management: Handling user authentication and session management
- Configuration Management: Managing model parameters and settings
- Conversation Management: Storing and organizing interaction history

### Generic Domains
- Analytics: Usage tracking and performance monitoring
- Billing: Usage calculation and payment processing

## Bounded Contexts

### Playground Context
Domain: Core interaction system for communicating with AI models

#### Aggregates:

1. Conversation
   - Root: ConversationRoot
   - Entities:
     - Message (prompt/response pairs)
     - ModelConfiguration
   - Value Objects:
     - PromptText
     - ResponseText
     - Timestamp
     - MessageMetadata

2. ModelSettings
   - Root: ModelSettingsRoot
   - Value Objects:
     - Temperature
     - MaxTokens
     - TopP
     - FrequencyPenalty
     - PresencePenalty
     - StopSequences

#### Domain Events:
- ConversationStarted
- MessageSent
- ResponseReceived
- SettingsUpdated
- ConversationSaved

#### Domain Services:
- ModelInteractionService
- PromptValidationService
- ResponseFormattingService

### User Context
Domain: User management and authentication

#### Aggregates:

1. User
   - Root: UserRoot
   - Entities:
     - UserProfile
     - ApiKeys
   - Value Objects:
     - Email
     - Password
     - UsageQuota

#### Domain Events:
- UserRegistered
- UserLoggedIn
- ApiKeyGenerated
- QuotaUpdated

#### Domain Services:
- AuthenticationService
- QuotaManagementService

### Configuration Context
Domain: Management of model configurations and preferences

#### Aggregates:

1. SavedConfiguration
   - Root: ConfigurationRoot
   - Value Objects:
     - ConfigName
     - ModelParameters
     - Preset

#### Domain Events:
- ConfigurationSaved
- ConfigurationApplied
- PresetCreated

#### Domain Services:
- ConfigurationManagementService
- PresetManagementService

## Tactical Design

### Value Objects

```typescript
class PromptText {
    private readonly text: string;
    
    constructor(text: string) {
        if (!this.isValid(text)) {
            throw new InvalidPromptError();
        }
        this.text = text;
    }
    
    private isValid(text: string): boolean {
        return text.length > 0 && text.length <= 32000;
    }
}

class Temperature {
    private readonly value: number;
    
    constructor(value: number) {
        if (value < 0 || value > 2) {
            throw new InvalidTemperatureError();
        }
        this.value = value;
    }
}
```

### Entities

```typescript
class Message {
    private readonly id: UUID;
    private readonly prompt: PromptText;
    private response: ResponseText;
    private readonly timestamp: Timestamp;
    private metadata: MessageMetadata;
    
    constructor(prompt: PromptText) {
        this.id = UUID.generate();
        this.prompt = prompt;
        this.timestamp = Timestamp.now();
    }
    
    updateResponse(response: ResponseText): void {
        this.response = response;
        this.metadata.updateResponseMetadata();
    }
}
```

### Aggregates

```typescript
class ConversationRoot {
    private readonly id: UUID;
    private readonly messages: Message[];
    private settings: ModelSettings;
    private status: ConversationStatus;
    
    startNewMessage(prompt: PromptText): Message {
        if (!this.canAddMessage()) {
            throw new ConversationLimitError();
        }
        
        const message = new Message(prompt);
        this.messages.push(message);
        return message;
    }
    
    updateSettings(settings: ModelSettings): void {
        this.settings = settings;
        this.domainEvents.push(new SettingsUpdated(this.id, settings));
    }
}
```

### Domain Services

```typescript
interface ModelInteractionService {
    sendPrompt(conversation: ConversationRoot, prompt: PromptText): Promise<ResponseText>;
    validateSettings(settings: ModelSettings): boolean;
    calculateTokenUsage(text: string): number;
}

class PromptValidationService {
    validate(prompt: PromptText, settings: ModelSettings): ValidationResult {
        // Implement validation logic
    }
}
```

## Implementation Guidelines

### Repository Interfaces

```typescript
interface ConversationRepository {
    save(conversation: ConversationRoot): Promise<void>;
    findById(id: UUID): Promise<ConversationRoot>;
    findByUser(userId: UUID): Promise<ConversationRoot[]>;
}

interface ConfigurationRepository {
    saveConfiguration(config: SavedConfiguration): Promise<void>;
    findByUser(userId: UUID): Promise<SavedConfiguration[]>;
}
```

### Application Services

```typescript
class PlaygroundApplicationService {
    constructor(
        private modelService: ModelInteractionService,
        private conversationRepo: ConversationRepository,
        private configRepo: ConfigurationRepository
    ) {}
    
    async startNewConversation(
        userId: UUID,
        settings: ModelSettings
    ): Promise<ConversationRoot> {
        const conversation = new ConversationRoot(userId, settings);
        await this.conversationRepo.save(conversation);
        return conversation;
    }
    
    async sendPrompt(
        conversationId: UUID,
        promptText: string
    ): Promise<ResponseText> {
        const conversation = await this.conversationRepo.findById(conversationId);
        const prompt = new PromptText(promptText);
        
        const message = conversation.startNewMessage(prompt);
        const response = await this.modelService.sendPrompt(conversation, prompt);
        
        message.updateResponse(response);
        await this.conversationRepo.save(conversation);
        
        return response;
    }
}
```

## Event Handling

### Domain Events

```typescript
class ConversationStarted {
    constructor(
        public readonly conversationId: UUID,
        public readonly userId: UUID,
        public readonly timestamp: Timestamp
    ) {}
}

class MessageSent {
    constructor(
        public readonly conversationId: UUID,
        public readonly messageId: UUID,
        public readonly prompt: PromptText,
        public readonly timestamp: Timestamp
    ) {}
}
```

### Event Handlers

```typescript
class ConversationEventHandler {
    @HandleEvent(ConversationStarted)
    async handleConversationStarted(event: ConversationStarted): Promise<void> {
        // Implement event handling logic
    }
    
    @HandleEvent(MessageSent)
    async handleMessageSent(event: MessageSent): Promise<void> {
        // Implement event handling logic
    }
}
```

## API Contracts

### REST Endpoints

```typescript
interface PlaygroundApi {
    // Conversations
    POST /api/conversations
    GET /api/conversations/{id}
    POST /api/conversations/{id}/messages
    
    // Configurations
    POST /api/configurations
    GET /api/configurations
    PUT /api/configurations/{id}
    
    // User Management
    POST /api/users/register
    POST /api/users/login
    GET /api/users/quota
}
```

### WebSocket Events

```typescript
interface WebSocketEvents {
    // Client -> Server
    SEND_PROMPT: {
        conversationId: string;
        prompt: string;
    }
    
    // Server -> Client
    RESPONSE_CHUNK: {
        conversationId: string;
        messageId: string;
        chunk: string;
    }
    
    RESPONSE_COMPLETE: {
        conversationId: string;
        messageId: string;
        metadata: MessageMetadata;
    }
}
```
