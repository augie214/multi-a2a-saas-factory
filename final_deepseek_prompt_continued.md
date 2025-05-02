## Continued Implementation Details

### Agent 2: Platform Sync Specialist

```javascript
const BaseAgent = require('../framework/BaseAgent');

class PlatformSyncSpecialist extends BaseAgent {
  constructor(id, config = {}) {
    super(id, 'Platform Sync Specialist', 'PlatformSyncSpecialist', config);
    this.connectedPlatforms = [];
    this.dataFlows = [];
    this.integrationMetrics = {
      totalDataTransferred: 0,
      successfulSyncs: 0,
      failedSyncs: 0,
      lastSyncTime: null
    };
  }
  
  async initializeCapabilities() {
    this.capabilities = [
      'platform_integration',
      'data_flow_management',
      'api_connectivity',
      'webhook_handling',
      'data_transformation',
      'sync_monitoring'
    ];
    
    // Initialize platform connectors
    this.platformConnectors = {
      notion: new NotionConnector(this.config.credentials?.notion),
      lovable: new LovableConnector(this.config.credentials?.lovable),
      manus: new ManusConnector(this.config.credentials?.manus),
      wordpress: new WordPressConnector(this.config.credentials?.wordpress),
      shopify: new ShopifyConnector(this.config.credentials?.shopify),
      youtube: new YouTubeConnector(this.config.credentials?.youtube),
      tiktok: new TikTokConnector(this.config.credentials?.tiktok),
      linkedin: new LinkedInConnector(this.config.credentials?.linkedin),
      facebook: new FacebookConnector(this.config.credentials?.facebook),
      instagram: new InstagramConnector(this.config.credentials?.instagram),
      etsy: new EtsyConnector(this.config.credentials?.etsy),
      amazon: new AmazonConnector(this.config.credentials?.amazon)
    };
    
    // Set up data flow monitoring
    this.dataFlowMonitor = new DataFlowMonitor(this.dataStore);
    
    // Schedule regular sync checks and reporting
    this.scheduler.scheduleHourly('syncCheck', this.checkSynchronization.bind(this));
    this.scheduler.scheduleDaily('syncReport', this.generateSyncReport.bind(this));
    
    // Restore existing connections and flows
    await this.restorePlatformConnections();
    await this.restoreDataFlows();
  }
  
  async processMessage(message) {
    switch (message.type) {
      case 'connect_platform':
        return this.connectToPlatform(message.platformName, message.credentials);
        
      case 'setup_data_flow':
        return this.setupDataFlow(message.sourceConfig, message.destinationConfig, message.transformationRules);
        
      case 'execute_data_flow':
        return this.executeDataFlow(message.dataFlowId);
        
      case 'check_platform_health':
        return this.checkPlatformHealth(message.platformName);
        
      case 'get_platform_capabilities':
        return this.getPlatformCapabilities(message.platformName);
        
      case 'update_credentials':
        return this.updatePlatformCredentials(message.platformName, message.credentials);
        
      default:
        this.logger.warn(`Unknown message type received: ${message.type}`);
        return { error: 'Unknown message type' };
    }
  }
  
  async connectToPlatform(platformName, credentials) {
    try {
      // Check if platform is supported
      if (!this.platformConnectors[platformName]) {
        throw new Error(`Platform ${platformName} not supported`);
      }
      
      const connector = this.platformConnectors[platformName];
      
      // Authenticate with platform
      await connector.authenticate(credentials);
      
      // Test connection
      const connectionTest = await connector.testConnection();
      if (!connectionTest.success) {
        throw new Error(`Connection test failed: ${connectionTest.error}`);
      }
      
      // Store connection info
      const connection = {
        id: generateUUID(),
        name: platformName,
        connectedAt: new Date(),
        status: 'active',
        capabilities: connector.getCapabilities()
      };
      
      this.connectedPlatforms.push(connection);
      await this.dataStore.savePlatformConnection(connection);
      
      // Log successful connection
      this.logger.info(`Connected to ${platformName} successfully`, { connectionId: connection.id });
      
      // Notify Chief Project Strategist
      await this.notifyAgent('ChiefProjectStrategist', {
        type: 'platform_connected',
        platform: platformName,
        connectionId: connection.id,
        capabilities: connection.capabilities
      });
      
      return { 
        success: true, 
        connectionId: connection.id,
        capabilities: connection.capabilities
      };
    } catch (error) {
      this.logger.error(`Error connecting to ${platformName}`, error);
      await this.logError('platformConnection', error);
      return { success: false, error: error.message };
    }
  }
  
  async setupDataFlow(sourceConfig, destinationConfig, transformationRules) {
    try {
      // Validate source and destination platforms
      const sourceConnector = this.platformConnectors[sourceConfig.platform];
      const destinationConnector = this.platformConnectors[destinationConfig.platform];
      
      if (!sourceConnector) {
        throw new Error(`Source platform ${sourceConfig.platform} not supported`);
      }
      
      if (!destinationConnector) {
        throw new Error(`Destination platform ${destinationConfig.platform} not supported`);
      }
      
      // Create data flow object
      const dataFlow = new DataFlow(
        sourceConnector,
        destinationConnector,
        transformationRules
      );
      
      // Validate data flow configuration
      await dataFlow.validate();
      
      // Initialize data flow
      await dataFlow.initialize();
      
      // Register data flow
      const flow = {
        id: generateUUID(),
        source: sourceConfig,
        destination: destinationConfig,
        transformationRules,
        status: 'active',
        createdAt: new Date(),
        lastExecuted: null,
        executionStats: {
          totalExecutions: 0,
          successfulExecutions: 0,
          failedExecutions: 0,
          averageExecutionTime: 0,
          totalRecordsProcessed: 0
        },
        passiveIncomeConfig: {
          enabled: true,
          estimationModel: this.determineIncomeModel(sourceConfig.platform, destinationConfig.platform),
          trackedMetrics: this.determineMetricsToTrack(sourceConfig.platform, destinationConfig.platform)
        }
      };
      
      this.dataFlows.push(flow);
      await this.dataStore.saveDataFlow(flow);
      
      // Schedule data flow execution
      this.scheduler.scheduleCustom(
        `dataFlow_${flow.id}`, 
        sourceConfig.frequency || '*/15 * * * *', // Default to every 15 minutes
        () => this.executeDataFlow(flow.id)
      );
      
      // Notify Chief Project Strategist
      await this.notifyAgent('ChiefProjectStrategist', {
        type: 'data_flow_created',
        dataFlowId: flow.id,
        source: sourceConfig.platform,
        destination: destinationConfig.platform
      });
      
      return { 
        success: true, 
        dataFlowId: flow.id 
      };
    } catch (error) {
      this.logger.error('Error setting up data flow', error);
      await this.logError('dataFlowSetup', error);
      return { success: false, error: error.message };
    }
  }
  
  async executeDataFlow(dataFlowId) {
    const dataFlow = this.dataFlows.find(df => df.id === dataFlowId);
    if (!dataFlow || dataFlow.status !== 'active') {
      return { success: false, error: 'Data flow not found or inactive' };
    }
    
    const startTime = Date.now();
    
    try {
      // Get connectors
      const sourceConnector = this.platformConnectors[dataFlow.source.platform];
      const destinationConnector = this.platformConnectors[dataFlow.destination.platform];
      
      // Extract data from source
      const sourceData = await sourceConnector.extractData(dataFlow.source.query);
      
      // Transform data
      const transformer = new DataTransformer(dataFlow.transformationRules);
      const transformedData = transformer.transform(sourceData);
      
      // Load data into destination
      const result = await destinationConnector.loadData(
        dataFlow.destination.target,
        transformedData
      );
      
      // Calculate execution time
      const executionTime = Date.now() - startTime;
      
      // Update statistics
      dataFlow.lastExecuted = new Date();
      dataFlow.executionStats.totalExecutions++;
      dataFlow.executionStats.successfulExecutions++;
      dataFlow.executionStats.totalRecordsProcessed += transformedData.length;
      
      // Update average execution time
      const totalExecutions = dataFlow.executionStats.totalExecutions;
      const currentAvg = dataFlow.executionStats.averageExecutionTime;
      dataFlow.executionStats.averageExecutionTime = 
        (currentAvg * (totalExecutions - 1) + executionTime) / totalExecutions;
      
      // Update integration metrics
      this.integrationMetrics.totalDataTransferred += transformedData.length;
      this.integrationMetrics.successfulSyncs++;
      this.integrationMetrics.lastSyncTime = new Date();
      
      // Log success
      await this.dataFlowMonitor.recordSuccess(dataFlowId, {
        records: transformedData.length,
        executionTime,
        timestamp: new Date()
      });
      
      // Save updated data flow
      await this.dataStore.updateDataFlow(dataFlowId, {
        lastExecuted: dataFlow.lastExecuted,
        executionStats: dataFlow.executionStats
      });
      
      // Calculate potential earnings
      const earnings = this.calculateDataFlowEarnings(dataFlow, transformedData);
      
      // Record earnings for passive income tracking
      if (dataFlow.passiveIncomeConfig.enabled && earnings.estimated > 0) {
        await this.dataStore.recordPassiveIncome({
          source: 'data_flow',
          dataFlowId: dataFlow.id,
          sourcePlatform: dataFlow.source.platform,
          destinationPlatform: dataFlow.destination.platform,
          amount: earnings.estimated,
          currency: earnings.currency,
          timestamp: new Date(),
          records: transformedData.length
        });
      }
      
      return { 
        success: true, 
        records: transformedData.length,
        executionTime,
        earnings
      };
    } catch (error) {
      // Update statistics
      dataFlow.lastExecuted = new Date();
      dataFlow.executionStats.totalExecutions++;
      dataFlow.executionStats.failedExecutions++;
      
      // Update integration metrics
      this.integrationMetrics.failedSyncs++;
      this.integrationMetrics.lastSyncTime = new Date();
      
      // Log failure
      this.logger.error(`Data flow execution failed: ${dataFlowId}`, error);
      await this.dataFlowMonitor.recordFailure(dataFlowId, {
        error: error.message,
        timestamp: new Date()
      });
      
      // Save updated data flow
      await this.dataStore.updateDataFlow(dataFlowId, {
        lastExecuted: dataFlow.lastExecuted,
        executionStats: dataFlow.executionStats
      });
      
      return { success: false, error: error.message };
    }
  }
  
  calculateDataFlowEarnings(dataFlow, transformedData) {
    // Calculate potential earnings based on data flow type
    let baseEarnings = 0;
    let revenueModel = 'generic';
    
    // Different earnings models based on destination platform
    switch (dataFlow.destination.platform) {
      case 'shopify':
        // Estimate based on product listings
        const products = transformedData.filter(d => d.type === 'product');
        baseEarnings = products.length * 5.50; // Average profit per product
        revenueModel = 'ecommerce';
        break;
        
      case 'wordpress':
        // Estimate based on content publishing (ad revenue)
        const posts = transformedData.filter(d => d.type === 'post');
        baseEarnings = posts.length * 2.75; // Average ad revenue per post
        revenueModel = 'content';
        break;
        
      case 'youtube':
        // Estimate based on video publishing
        const videos = transformedData.filter(d => d.type === 'video');
        baseEarnings = videos.length * 15.50; // Average revenue per video
        revenueModel = 'video';
        break;
        
      case 'tiktok':
        // Estimate based on content publishing
        const tiktokPosts = transformedData.filter(d => d.type === 'post');
        baseEarnings = tiktokPosts.length * 8.25; // Average revenue per post
        revenueModel = 'social';
        break;
        
      case 'etsy':
        // Estimate based on product listings
        const etsyProducts = transformedData.filter(d => d.type === 'product');
        baseEarnings = etsyProducts.length * 7.25; // Average profit per product
        revenueModel = 'marketplace';
        break;
        
      default:
        // Generic estimation
        baseEarnings = transformedData.length * 0.50;
        revenueModel = 'data';
    }
    
    return {
      estimated: parseFloat(baseEarnings.toFixed(2)),
      currency: 'USD',
      model: revenueModel,
      source: dataFlow.destination.platform,
      timestamp: new Date()
    };
  }
  
  determineIncomeModel(sourcePlatform, destinationPlatform) {
    // Determine which income model to use based on the platforms
    if (destinationPlatform === 'shopify' || destinationPlatform === 'etsy' || destinationPlatform === 'amazon') {
      return 'ecommerce';
    } else if (destinationPlatform === 'youtube' || destinationPlatform === 'tiktok') {
      return 'content_monetization';
    } else if (destinationPlatform === 'wordpress' || destinationPlatform === 'medium') {
      return 'ad_revenue';
    } else {
      return 'generic';
    }
  }
  
  determineMetricsToTrack(sourcePlatform, destinationPlatform) {
    // Determine which metrics to track for income calculation
    const baseMetrics = ['records_processed', 'execution_time'];
    
    if (destinationPlatform === 'shopify' || destinationPlatform === 'etsy' || destinationPlatform === 'amazon') {
      return [
        ...baseMetrics,
        'products_created',
        'products_updated',
        'sales_generated',
        'revenue_generated'
      ];
    } else if (destinationPlatform === 'youtube' || destinationPlatform === 'tiktok') {
      return [
        ...baseMetrics,
        'videos_created',
        'views_generated',
        'engagement_rate',
        'ad_impressions'
      ];
    } else if (destinationPlatform === 'wordpress' || destinationPlatform === 'medium') {
      return [
        ...baseMetrics,
        'posts_created',
        'visitors',
        'page_views',
        'ad_clicks'
      ];
    } else {
      return baseMetrics;
    }
  }
}

module.exports = PlatformSyncSpecialist;
```

### Agent 3: Testing Tool Analyst

```javascript
const BaseAgent = require('../framework/BaseAgent');

class TestingToolAnalyst extends BaseAgent {
  constructor(id, config = {}) {
    super(id, 'Testing Tool Analyst', 'TestingToolAnalyst', config);
    this.toolRegistry = new TestingToolRegistry();
    this.testingFrameworks = [];
    this.testResults = {};
  }
  
  async initializeCapabilities() {
    this.capabilities = [
      'testing_tool_evaluation',
      'test_automation',
      'quality_assurance',
      'benchmark_analysis',
      'tool_selection',
      'test_framework_configuration'
    ];
    
    // Initialize tool research system
    this.toolResearcher = new ToolResearcher();
    
    // Set up evaluation metrics
    this.evaluationMetrics = [
      { name: 'speed', weight: 0.3, description: 'Execution time of standard test suite' },
      { name: 'coverage', weight: 0.25, description: 'Percentage of code/functionality covered' },
      { name: 'reliability', weight: 0.2, description: 'Consistency of test results' },
      { name: 'easeOfUse', weight: 0.15, description: 'Learning curve and configuration complexity' },
      { name: 'communitySupport', weight: 0.1, description: 'Activity level and size of community' }
    ];
    
    // Schedule regular tool evaluation
    this.scheduler.scheduleWeekly('toolEvaluation', this.evaluateTestingTools.bind(this));
    this.scheduler.scheduleDaily('runTests', this.runDailyTests.bind(this));
    
    // Initialize benchmark suite
    this.benchmarkSuite = new BenchmarkSuite();
    
    // Load initial tool registry
    await this.initializeToolRegistry();
  }
  
  async processMessage(message) {
    switch (message.type) {
      case 'research_tools':
        return this.researchTestingTools(message.category);
        
      case 'evaluate_tool':
        return this.evaluateSingleTool(message.toolId);
        
      case 'run_tests':
        return this.runTests(message.scope, message.options);
        
      case 'get_test_results':
        return this.getTestResults(message.testRunId);
        
      case 'configure_framework':
        return this.configureTestingFramework(message.category, message.configuration);
        
      case 'generate_roi_report':
        return this.generateTestingROIReport();
        
      default:
        this.logger.warn(`Unknown message type received: ${message.type}`);
        return { error: 'Unknown message type' };
    }
  }
  
  async initializeToolRegistry() {
    // Register initial set of free, open-source testing tools
    const initialTools = [
      {
        name: 'TestSprite',
        description: 'Fast, lightweight testing framework for web applications',
        category: 'ui',
        license: 'MIT',
        url: 'https://github.com/testsprite/testsprite',
        stars: 2450,
        lastUpdated: new Date(Date.now() - 30 * 24 * 60 * 60 * 1000) // 30 days ago
      },
      {
        name: 'Cypress',
        description: 'End-to-end testing framework for web applications',
        category: 'ui',
        license: 'MIT',
        url: 'https://github.com/cypress-io/cypress',
        stars: 42000,
        lastUpdated: new Date(Date.now() - 5 * 24 * 60 * 60 * 1000) // 5 days ago
      },
      {
        name: 'Jest',
        description: 'JavaScript testing framework with a focus on simplicity',
        category: 'unit',
        license: 'MIT',
        url: 'https://github.com/facebook/jest',
        stars: 41000,
        lastUpdated: new Date(Date.now() - 2 * 24 * 60 * 60 * 1000) // 2 days ago
      },
      {
        name: 'Mocha',
        description: 'Flexible JavaScript test framework for Node.js and browsers',
        category: 'unit',
        license: 'MIT',
        url: 'https://github.com/mochajs/mocha',
        stars: 21000,
        lastUpdated: new Date(Date.now() - 10 * 24 * 60 * 60 * 1000) // 10 days ago
      },
      {
        name: 'Playwright',
        description: 'Browser automation library for end-to-end testing',
        category: 'ui',
        license: 'Apache-2.0',
        url: 'https://github.com/microsoft/playwright',
        stars: 35000,
        lastUpdated: new Date(Date.now() - 1 * 24 * 60 * 60 * 1000) // 1 day ago
      },
      {
        name: 'Supertest',
        description: 'Super-agent driven library for testing HTTP servers',
        category: 'api',
        license: 'MIT',
        url: 'https://github.com/visionmedia/supertest',
        stars: 10000,
        lastUpdated: new Date(Date.now() - 45 * 24 * 60 * 60 * 1000) // 45 days ago
      },
      {
        name: 'JMeter',
        description: 'Load testing and performance measurement tool',
        category: 'performance',
        license: 'Apache-2.0',
        url: 'https://github.com/apache/jmeter',
        stars: 6000,
        lastUpdated: new Date(Date.now() - 15 * 24 * 60 * 60 * 1000) // 15 days ago
      },
      {
        name: 'Lighthouse',
        description: 'Automated tool for improving web page quality',
        category: 'performance',
        license: 'Apache-2.0',
        url: 'https://github.com/GoogleChrome/lighthouse',
        stars: 25000,
        lastUpdated: new Date(Date.now() - 7 * 24 * 60 * 60 * 1000) // 7 days ago
      }
    ];
    
    for (const tool of initialTools) {
      await this.toolRegistry.registerTool({
        id: generateUUID(),
        ...tool,
        registeredAt: new Date()
      });
    }
    
    this.logger.info(`Initialized tool registry with ${initialTools.length} tools`);
  }
  
  async researchTestingTools(category) {
    try {
      this.logger.info(`Researching testing tools for category: ${category}`);
      
      const toolOptions = await this.toolResearcher.findOpenSourceTools({
        category,
        licenseTypes: ['MIT', 'Apache-2.0', 'BSD-3-Clause'],
        minStars: 100
      });
      
      // Register newly discovered tools
      const newTools = [];
      for (const tool of toolOptions) {
        // Check if tool already exists
        const existingTool = await this.toolRegistry.findToolByName(tool.name);
        
        if (!existingTool) {
          const newTool = {
            id: generateUUID(),
            ...tool,
            registeredAt: new Date()
          };
          
          await this.toolRegistry.registerTool(newTool);
          newTools.push(newTool);
        }
      }
      
      this.logger.info(`Discovered ${newTools.length} new testing tools for category: ${category}`);
      
      // Notify Chief Project Strategist about new tools
      if (newTools.length > 0) {
        await this.notifyAgent('ChiefProjectStrategist', {
          type: 'new_tools_discovered',
          category,
          toolCount: newTools.length,
          tools: newTools.map(t => ({
            name: t.name,
            description: t.description,
            stars: t.stars,
            license: t.license
          }))
        });
      }
      
      return {
        success: true,
        category,
        toolsFound: toolOptions.length,
        newToolsRegistered: newTools.length,
        tools: toolOptions.map(tool => ({
          name: tool.name,
          description: tool.description,
          license: tool.license,
          stars: tool.stars,
          url: tool.url
        }))
      };
    } catch (error) {
      this.logger.error(`Error researching testing tools for category: ${category}`, error);
      return { success: false, error: error.message };
    }
  }
  
  async evaluateTestingTools() {
    try {
      const categories = ['ui', 'api', 'unit', 'integration', 'performance'];
      const evaluationResults = {};
      
      for (const category of categories) {
        this.logger.info(`Evaluating testing tools for category: ${category}`);
        
        // Get tools for this category
        const tools = await this.toolRegistry.getToolsByCategory(category);
        
        if (tools.length === 0) {
          this.logger.info(`No tools found for category: ${category}`);
          evaluationResults[category] = {
            message: `No tools found for category: ${category}`,
            timestamp: new Date()
          };
          continue;
        }
        
        // Run benchmarks for each tool
        const benchmarkResults = await this.runBenchmarks(tools);
        
        // Score tools based on benchmark results
        const scoredTools = tools.map(tool => {
          const toolBenchmarks = benchmarkResults[tool.id];
          
          if (!toolBenchmarks) {
            return {
              ...tool,
              score: 0,
              benchmarks: null,
              error: 'Benchmark failed'
            };
          }
          
          // Calculate weighted score
          let totalScore = 0;
          for (const metric of this.evaluationMetrics) {
            const normalizedScore = this.normalizeScore(
              toolBenchmarks[metric.name],
              metric.name
            );
            totalScore += normalizedScore * metric.weight;
          }
          
          return {
            ...tool,
            score: totalScore,
            benchmarks: toolBenchmarks
          };
        });
        
        // Sort by score and select top tool
        scoredTools.sort((a, b) => b.score - a.score);
        const validTools = scoredTools.filter(tool => tool.benchmarks !== null);
        
        if (validTools.length === 0) {
          this.logger.warn(`No valid benchmark results for category: ${category}`);
          evaluationResults[category] = {
            error: 'No valid benchmark results',
            timestamp: new Date()
          };
          continue;
        }
        
        const selectedTool = validTools[0];
        
        evaluationResults[category] = {
          selectedTool,
          allTools: scoredTools,
          evaluationDate: new Date()
        };
        
        // Update testing framework if needed
        await this.updateTestingFramework(category, selectedTool);
      }
      
      // Save evaluation results
      await this.dataStore.saveTestingToolEvaluation({
        id: generateUUID(),
        results: evaluationResults,
        timestamp: new Date()
      });
      
      // Notify Chief Project Strategist
      await this.notifyAgent('ChiefProjectStrategist', {
        type: 'testing_tool_evaluation',
        evaluationResults: Object.entries(evaluationResults).map(([category, result]) => ({
          category,
          selectedTool: result.selectedTool ? {
            name: result.selectedTool.name,
            score: result.selectedTool.score
          } : null,
          evaluationDate: result.evaluationDate
        }))
      });
      
      return {
        success: true,
        evaluationResults
      };
    } catch (error) {
      this.logger.error('Error evaluating testing tools', error);
      return { success: false, error: error.message };
    }
  }
  
  async runBenchmarks(tools) {
    try {
      const results = {};
      
      for (const tool of tools) {
        this.logger.info(`Running benchmarks for tool: ${tool.name}`);
        
        try {
          // Create isolated testing environment
          const testEnv = await this.createTestEnvironment();
          
          // Install tool
          await testEnv.installTool(tool);
          
          // Run standard benchmark suite
          const benchmarkResults = await testEnv.runBenchmarkSuite(tool);
          
          results[tool.id] = benchmarkResults;
          
          // Clean up test environment
          await testEnv.cleanup();
        } catch (error) {
          this.logger.error(`Error benchmarking tool: ${tool.name}`, error);
          results[tool.id] = null;
        }
      }
      
      return results;
    } catch (error) {
      this.logger.error('Error running benchmarks', error);
      throw error;
    }
  }
  
  async createTestEnvironment() {
    // Create isolated environment for testing
    const env = new TestEnvironment();
    await env.initialize();
    return env;
  }
  
  normalizeScore(rawScore, metricName) {
    // Different metrics have different normalization approaches
    switch (metricName) {
      case 'speed':
        // Lower is better for speed (execution time)
        return 1 - (rawScore / 100); // Assuming 100s is the worst acceptable time
      
      case 'coverage':
        // Higher is better for coverage (percentage)
        return rawScore / 100;
      
      case 'reliability':
        // Higher is better for reliability (percentage of passed tests)
        return rawScore / 100;
      
      case 'easeOfUse':
        // Higher is better for ease of use (rating 1-5)
        return rawScore / 5;
      
      case 'communitySupport':
        // Higher is better for community support (composite score)
        return rawScore / 10;
      
      default:
        return rawScore;
    }
  }
  
  async updateTestingFramework(category, selectedTool) {
    try {
      const index = this.testingFrameworks.findIndex(tf => tf.category === category);
      
      if (index >= 0) {
        const currentFramework = this.testingFrameworks[index];
        
        // Only update if different tool selected or significant version change
        if (currentFramework.toolId !== selectedTool.id || 
            currentFramework.version !== selectedTool.version) {
          
          // Update existing framework
          this.testingFrameworks[index] = {
            category,
            toolId: selectedTool.id,
            toolName: selectedTool.name,
            version: selectedTool.version,
            updatedAt: new Date(),
            previousToolId: currentFramework.toolId
          };
          
          this.logger.info(`Updated testing framework for ${category} from ${currentFramework.toolName} to ${selectedTool.name}`);
        }
      } else {
        // Add new framework
        this.testingFrameworks.push({
          category,