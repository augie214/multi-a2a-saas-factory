# Multi-Agenic Autonomous Agent SaaS AI Factory
## Complete Implementation Blueprint

### Executive Overview
You are tasked with developing a Fortune 500-caliber Multi-Agenic Autonomous Agent SaaS AI Factory. This platform will democratize enterprise-grade digital and SaaS products by deploying a dynamic ecosystem of AI agents that create, manage, and optimize phenomenal solutions accessible to users of all socioeconomic backgrounds and technical skill levels.

The platform prioritizes:
- **Accessibility**: All users, regardless of wealth or technical expertise, can leverage enterprise-grade AI
- **Analytics-driven**: All decisions based on data, not assumptions or preferences
- **Autonomous operation**: Agents continuously generate passive income with minimal user input
- **Fortune 500-level**: Enterprise-grade reliability, security, and scalability

## System Architecture

### 1. Core Platform Infrastructure
```javascript
// system-architecture.js
const express = require('express');
const { ApolloServer } = require('apollo-server-express');
const mongoose = require('mongoose');
const amqp = require('amqplib');
const { typeDefs, resolvers } = require('./graphql');
const { authenticateJWT } = require('./middleware/auth');
const winston = require('winston');

class SystemArchitecture {
  constructor(config) {
    this.config = config;
    this.app = express();
    this.logger = this.setupLogger();
    this.messageBus = null;
    this.agentRegistry = new AgentRegistry();
  }
  
  async initialize() {
    try {
      // Connect to MongoDB
      await mongoose.connect(this.config.mongodb.uri, {
        useNewUrlParser: true,
        useUnifiedTopology: true,
        useFindAndModify: false,
        useCreateIndex: true
      });
      this.logger.info('Connected to MongoDB');
      
      // Connect to RabbitMQ message bus
      const connection = await amqp.connect(this.config.rabbitmq.uri);
      this.messageBus = await connection.createChannel();
      await this.messageBus.assertExchange('agent-communication', 'topic', { durable: true });
      this.logger.info('Connected to RabbitMQ');
      
      // Initialize Express middleware
      this.app.use(express.json());
      this.app.use(authenticateJWT);
      
      // Set up GraphQL server
      const server = new ApolloServer({
        typeDefs,
        resolvers,
        context: ({ req }) => ({ user: req.user, agentRegistry: this.agentRegistry })
      });
      await server.start();
      server.applyMiddleware({ app: this.app });
      
      // Set up REST endpoints
      this.setupRESTEndpoints();
      
      // Initialize agent registry
      await this.agentRegistry.initialize();
      
      // Start HTTP server
      const port = this.config.server.port || 3000;
      this.app.listen(port, () => {
        this.logger.info(`Server running on port ${port}`);
      });
      
      return { success: true };
    } catch (error) {
      this.logger.error('System initialization failed', error);
      throw error;
    }
  }
  
  setupLogger() {
    return winston.createLogger({
      level: this.config.logging.level || 'info',
      format: winston.format.combine(
        winston.format.timestamp(),
        winston.format.json()
      ),
      transports: [
        new winston.transports.Console(),
        new winston.transports.File({ 
          filename: 'error.log', 
          level: 'error' 
        }),
        new winston.transports.File({ 
          filename: 'combined.log' 
        })
      ]
    });
  }
  
  setupRESTEndpoints() {
    // Health check endpoint
    this.app.get('/health', (req, res) => {
      res.status(200).json({ status: 'ok' });
    });
    
    // Agent API endpoints
    this.app.use('/api/agents', require('./routes/agents'));
    
    // Dashboard API endpoints
    this.app.use('/api/dashboard', require('./routes/dashboard'));
    
    // Platform integration endpoints
    this.app.use('/api/integrations', require('./routes/integrations'));
    
    // User management endpoints
    this.app.use('/api/users', require('./routes/users'));
    
    // Storage management endpoints
    this.app.use('/api/storage', require('./routes/storage'));
    
    // Error handling middleware
    this.app.use((err, req, res, next) => {
      this.logger.error('API Error', { error: err.message, stack: err.stack });
      res.status(err.status || 500).json({
        error: {
          message: err.message,
          status: err.status || 500
        }
      });
    });
  }
  
  async shutdown() {
    try {
      // Close MongoDB connection
      await mongoose.connection.close();
      this.logger.info('MongoDB connection closed');
      
      // Close RabbitMQ connection
      if (this.messageBus) {
        await this.messageBus.close();
        this.logger.info('RabbitMQ connection closed');
      }
      
      this.logger.info('System shutdown complete');
      return { success: true };
    } catch (error) {
      this.logger.error('System shutdown failed', error);
      throw error;
    }
  }
}

module.exports = SystemArchitecture;
```

### 2. Base Agent Framework
```javascript
// base-agent.js
const EventEmitter = require('events');
const { v4: uuidv4 } = require('uuid');

class BaseAgent extends EventEmitter {
  constructor(id, name, type, config = {}) {
    super();
    this.id = id || uuidv4();
    this.name = name;
    this.type = type;
    this.config = config;
    this.status = 'created';
    this.capabilities = [];
    this.createdAt = new Date();
    this.lastActive = null;
    this.dependencies = [];
    this.metrics = {};
    this.logger = null;
    this.dataStore = null;
    this.messageBus = null;
    this.scheduler = null;
  }
  
  async initialize(services) {
    try {
      // Set up agent services
      this.logger = services.logger;
      this.dataStore = services.dataStore;
      this.messageBus = services.messageBus;
      this.scheduler = services.scheduler;
      this.agentRegistry = services.agentRegistry;
      
      // Subscribe to relevant message topics
      await this.setupMessageSubscriptions();
      
      // Initialize agent capabilities
      await this.initializeCapabilities();
      
      // Register with agent registry
      await this.registerWithRegistry();
      
      // Update status
      this.status = 'initialized';
      this.lastActive = new Date();
      
      this.logger.info(`Agent ${this.id} (${this.name}) initialized`);
      this.emit('initialized', { agentId: this.id });
      
      return {
        success: true,
        agentId: this.id,
        status: this.status
      };
    } catch (error) {
      this.status = 'error';
      this.logger.error(`Agent ${this.id} initialization failed`, error);
      this.emit('error', { agentId: this.id, error: error.message });
      throw error;
    }
  }
  
  async setupMessageSubscriptions() {
    // Subscribe to direct messages
    await this.messageBus.assertQueue(`agent.${this.id}`, { durable: true });
    this.messageBus.consume(`agent.${this.id}`, this.handleDirectMessage.bind(this));
    
    // Subscribe to broadcast messages
    await this.messageBus.assertExchange('agent-broadcast', 'fanout', { durable: true });
    const q = await this.messageBus.assertQueue('', { exclusive: true });
    await this.messageBus.bindQueue(q.queue, 'agent-broadcast', '');
    this.messageBus.consume(q.queue, this.handleBroadcastMessage.bind(this));
    
    // Subscribe to specific topics based on agent capabilities
    for (const capability of this.capabilities) {
      await this.messageBus.assertQueue(`capability.${capability}`, { durable: true });
      this.messageBus.consume(`capability.${capability}`, this.handleCapabilityMessage.bind(this));
    }
  }
  
  async handleDirectMessage(msg) {
    if (!msg) return;
    
    try {
      const content = JSON.parse(msg.content.toString());
      this.logger.debug(`Agent ${this.id} received direct message`, content);
      
      // Handle the message based on its type
      const result = await this.processMessage(content);
      
      // Acknowledge message processing
      this.messageBus.ack(msg);
      
      // If a reply is requested, send it
      if (content.replyTo) {
        await this.sendMessage(content.replyTo, {
          type: 'reply',
          requestId: content.requestId,
          result
        });
      }
      
      // Update last active timestamp
      this.lastActive = new Date();
    } catch (error) {
      this.logger.error(`Error handling message in agent ${this.id}`, error);
      // Reject the message for reprocessing if needed
      this.messageBus.nack(msg, false, true);
    }
  }
  
  async handleBroadcastMessage(msg) {
    if (!msg) return;
    
    try {
      const content = JSON.parse(msg.content.toString());
      this.logger.debug(`Agent ${this.id} received broadcast message`, content);
      
      // Process broadcast message
      await this.processBroadcastMessage(content);
      
      // Acknowledge message
      this.messageBus.ack(msg);
      
      // Update last active timestamp
      this.lastActive = new Date();
    } catch (error) {
      this.logger.error(`Error handling broadcast message in agent ${this.id}`, error);
      this.messageBus.ack(msg); // Acknowledge anyway to avoid infinite loop
    }
  }
  
  async handleCapabilityMessage(msg) {
    if (!msg) return;
    
    try {
      const content = JSON.parse(msg.content.toString());
      const capability = msg.fields.routingKey.split('.')[1];
      
      this.logger.debug(`Agent ${this.id} received capability message for ${capability}`, content);
      
      // Process capability-specific message
      await this.processCapabilityMessage(capability, content);
      
      // Acknowledge message
      this.messageBus.ack(msg);
      
      // Update last active timestamp
      this.lastActive = new Date();
    } catch (error) {
      this.logger.error(`Error handling capability message in agent ${this.id}`, error);
      this.messageBus.nack(msg, false, true);
    }
  }
  
  async sendMessage(recipient, message) {
    try {
      const fullMessage = {
        ...message,
        sender: this.id,
        timestamp: new Date().toISOString(),
        messageId: uuidv4()
      };
      
      await this.messageBus.sendToQueue(
        `agent.${recipient}`,
        Buffer.from(JSON.stringify(fullMessage)),
        {
          persistent: true,
          contentType: 'application/json'
        }
      );
      
      return { success: true, messageId: fullMessage.messageId };
    } catch (error) {
      this.logger.error(`Error sending message from agent ${this.id} to ${recipient}`, error);
      throw error;
    }
  }
  
  async broadcastMessage(message) {
    try {
      const broadcastMessage = {
        ...message,
        sender: this.id,
        timestamp: new Date().toISOString(),
        messageId: uuidv4()
      };
      
      await this.messageBus.publish(
        'agent-broadcast',
        '',
        Buffer.from(JSON.stringify(broadcastMessage)),
        {
          persistent: true,
          contentType: 'application/json'
        }
      );
      
      return { success: true, messageId: broadcastMessage.messageId };
    } catch (error) {
      this.logger.error(`Error broadcasting message from agent ${this.id}`, error);
      throw error;
    }
  }
  
  async notifyAgent(agentId, notification) {
    return this.sendMessage(agentId, {
      type: 'notification',
      notification
    });
  }
  
  async registerWithRegistry() {
    await this.agentRegistry.registerAgent({
      id: this.id,
      name: this.name,
      type: this.type,
      status: this.status,
      capabilities: this.capabilities,
      createdAt: this.createdAt,
      lastActive: this.lastActive
    });
  }
  
  // Abstract methods to be implemented by specific agents
  async initializeCapabilities() {
    throw new Error('Method not implemented');
  }
  
  async processMessage(message) {
    throw new Error('Method not implemented');
  }
  
  async processBroadcastMessage(message) {
    throw new Error('Method not implemented');
  }
  
  async processCapabilityMessage(capability, message) {
    throw new Error('Method not implemented');
  }
  
  // Common agent methods
  async getStatus() {
    return {
      id: this.id,
      name: this.name,
      type: this.type,
      status: this.status,
      capabilities: this.capabilities,
      createdAt: this.createdAt,
      lastActive: this.lastActive,
      metrics: this.metrics
    };
  }
  
  async updateConfiguration(config) {
    // Merge new configuration with existing
    this.config = {
      ...this.config,
      ...config
    };
    
    // Save updated configuration
    await this.dataStore.updateAgentConfig(this.id, this.config);
    
    // Emit configuration update event
    this.emit('configUpdated', { agentId: this.id });
    
    return { success: true };
  }
  
  async getSelfAssessedStrengths() {
    // Default implementation - should be overridden in specific agents
    return [
      {
        name: 'reliability',
        score: 0.9,
        evidence: 'Consistently operational with 99.9% uptime'
      },
      {
        name: 'efficiency',
        score: 0.85,
        evidence: 'Average task completion time improved by 15% over last week'
      }
    ];
  }
  
  async getProgressReport() {
    // Default implementation - should be overridden in specific agents
    return {
      tasks: [
        {
          name: 'System maintenance',
          status: 'completed',
          completedAt: new Date(Date.now() - 3600000)
        }
      ],
      metrics: [
        {
          name: 'Tasks completed',
          value: 25,
          target: 20,
          unit: 'count'
        }
      ]
    };
  }
  
  async receiveFeedback(feedback) {
    this.logger.info(`Agent ${this.id} received feedback`, feedback);
    // Store feedback for learning and improvement
    await this.dataStore.storeAgentFeedback(this.id, feedback);
    return { success: true };
  }
  
  async adjustStrategy(adjustments) {
    this.logger.info(`Agent ${this.id} adjusting strategy`, adjustments);
    // Default implementation - should be overridden
    return { success: true, applied: false, message: 'Strategy adjustment not implemented' };
  }
  
  async shutdown() {
    try {
      this.status = 'shutting_down';
      
      // Close message subscriptions
      // Clean up scheduled tasks
      await this.scheduler.removeAllTasksForAgent(this.id);
      
      // Update registry
      await this.agentRegistry.updateAgent(this.id, {
        status: 'offline',
        lastActive: new Date()
      });
      
      this.status = 'offline';
      this.emit('shutdown', { agentId: this.id });
      
      return { success: true };
    } catch (error) {
      this.logger.error(`Error shutting down agent ${this.id}`, error);
      throw error;
    }
  }
}

module.exports = BaseAgent;
```

### 3. Dashboard Implementation
```javascript
// Dashboard React Component
import React, { useState, useEffect } from 'react';
import { AgentCard } from './components/AgentCard';
import { AgentMetrics } from './components/AgentMetrics';
import { PassiveIncomeWidget } from './components/PassiveIncomeWidget';
import { AgentCreationModal } from './components/AgentCreationModal';
import { SystemHealth } from './components/SystemHealth';
import { MarketTrendsWidget } from './components/MarketTrendsWidget';
import { useAgentData } from './hooks/useAgentData';
import { useAuth } from './hooks/useAuth';
import './Dashboard.css';

export const Dashboard = () => {
  const { user } = useAuth();
  const { agents, loading, error, refreshAgents } = useAgentData();
  const [selectedAgent, setSelectedAgent] = useState(null);
  const [showCreateModal, setShowCreateModal] = useState(false);
  const [dashboardMetrics, setDashboardMetrics] = useState(null);
  const [viewMode, setViewMode] = useState('grid'); // 'grid' or 'list'
  
  useEffect(() => {
    // Fetch dashboard metrics
    const fetchMetrics = async () => {
      try {
        const response = await fetch('/api/dashboard/metrics', {
          headers: {
            'Authorization': `Bearer ${localStorage.getItem('token')}`
          }
        });
        const data = await response.json();
        setDashboardMetrics(data);
      } catch (error) {
        console.error('Failed to fetch dashboard metrics', error);
      }
    };
    
    fetchMetrics();
    
    // Set up real-time updates via WebSocket
    const socket = new WebSocket(`ws://${window.location.host}/ws/dashboard`);
    
    socket.onmessage = (event) => {
      const data = JSON.parse(event.data);
      
      if (data.type === 'agent_update') {
        refreshAgents();
      } else if (data.type === 'metrics_update') {
        setDashboardMetrics(prevMetrics => ({
          ...prevMetrics,
          ...data.metrics
        }));
      }
    };
    
    socket.onerror = (error) => {
      console.error('WebSocket error', error);
    };
    
    return () => {
      socket.close();
    };
  }, [refreshAgents]);
  
  const handleAgentClick = (agent) => {
    setSelectedAgent(agent);
  };
  
  const handleCreateAgent = () => {
    setShowCreateModal(true);
  };
  
  const handleViewModeChange = (mode) => {
    setViewMode(mode);
  };
  
  if (loading) return <div className="loading-container">Loading your AI agent factory...</div>;
  if (error) return <div className="error-container">Error: {error.message}</div>;
  
  return (
    <div className="dashboard">
      <header className="dashboard-header">
        <div className="header-title">
          <h1>Multi-Agenic Autonomous Agent Factory</h1>
          <span className="user-welcome">Welcome, {user.name}</span>
        </div>
        <div className="header-actions">
          <div className="view-toggle">
            <button 
              className={viewMode === 'grid' ? 'active' : ''} 
              onClick={() => handleViewModeChange('grid')}
            >
              <i className="fas fa-th"></i>
            </button>
            <button 
              className={viewMode === 'list' ? 'active' : ''} 
              onClick={() => handleViewModeChange('list')}
            >
              <i className="fas fa-list"></i>
            </button>
          </div>
          <button onClick={handleCreateAgent} className="create-agent-btn">
            <i className="fas fa-plus"></i> Deploy New Agent
          </button>
        </div>
      </header>
      
      <div className="dashboard-overview">
        <div className="overview-row">
          <PassiveIncomeWidget 
            passiveIncome={dashboardMetrics?.passiveIncome || {}} 
          />
          <SystemHealth 
            health={dashboardMetrics?.systemHealth || {}} 
          />
        </div>
        
        <div className="agent-metrics-container">
          <AgentMetrics agents={agents} />
        </div>
        
        <MarketTrendsWidget 
          trends={dashboardMetrics?.marketTrends || []} 
        />
      </div>
      
      <div className={`agents-container ${viewMode}`}>
        <h2>Your Agent Workforce</h2>
        <div className={`agents-${viewMode}`}>
          {agents.map(agent => (
            <AgentCard 
              key={agent.id}
              agent={agent}
              viewMode={viewMode}
              onClick={() => handleAgentClick(agent)}
            />
          ))}
        </div>
      </div>
      
      {selectedAgent && (
        <AgentDetailModal 
          agent={selectedAgent} 
          onClose={() => setSelectedAgent(null)}
        />
      )}
      
      {showCreateModal && (
        <AgentCreationModal 
          onClose={() => setShowCreateModal(false)}
          onAgentCreated={refreshAgents}
        />
      )}
    </div>
  );
};

// PassiveIncomeWidget Component
const PassiveIncomeWidget = ({ passiveIncome }) => {
  const {
    daily = 0,
    weekly = 0,
    monthly = 0,
    trends = []
  } = passiveIncome;
  
  return (
    <div className="widget passive-income-widget">
      <h3>Passive Income Dashboard</h3>
      <div className="income-summary">
        <div className="income-card">
          <h4>Today</h4>
          <p className="income-value">${daily.toFixed(2)}</p>
        </div>
        <div className="income-card">
          <h4>This Week</h4>
          <p className="income-value">${weekly.toFixed(2)}</p>
        </div>
        <div className="income-card">
          <h4>This Month</h4>
          <p className="income-value">${monthly.toFixed(2)}</p>
        </div>
      </div>
      <div className="income-chart">
        {/* Income trend chart would go here - using recharts or similar */}
      </div>
      <div className="income-sources">
        <h4>Income Sources</h4>
        <ul>
          {trends.map((source, index) => (
            <li key={index} className="income-source">
              <span className="source-name">{source.name}</span>
              <span className="source-amount">${source.amount.toFixed(2)}</span>
              <span className={`source-trend ${source.trend > 0 ? 'positive' : source.trend < 0 ? 'negative' : 'neutral'}`}>
                {source.trend > 0 ? '↑' : source.trend < 0 ? '↓' : '→'} 
                {Math.abs(source.trend)}%
              </span>
            </li>
          ))}
        </ul>
      </div>
    </div>
  );
};
```

## Detailed Agent Implementations

### 1. Chief Project Strategist
```javascript
const BaseAgent = require('../framework/BaseAgent');

class ChiefProjectStrategist extends BaseAgent {
  constructor(id, config = {}) {
    super(id, 'Chief Project Strategist', 'ChiefProjectStrategist', config);
    this.missionStatement = "Deliver phenomenal digital and SaaS products that are outstanding and accessible to all people—rich, poor, technical, or non-technical—without barriers";
    this.managedAgents = [];
    this.incomeOpportunities = [];
    this.strategicPlan = null;
  }
  
  async initializeCapabilities() {
    this.capabilities = [
      'project_management',
      'strategic_planning',
      'performance_evaluation',
      'market_analysis',
      'income_opportunity_analysis',
      'agent_coordination'
    ];
    
    // Initialize strategic analytics systems
    this.marketAnalytics = new MarketAnalyticsSystem({
      dataStore: this.dataStore,
      refreshInterval: 3600 // Hourly updates
    });
    
    this.incomeOpportunityAnalyzer = new IncomeOpportunityAnalyzer({
      marketAnalytics: this.marketAnalytics,
      minimumROI: 2.5, // 250% ROI minimum
      riskTolerance: this.config.riskTolerance || 'adaptive'
    });
    
    this.reportGenerator = new ReportGenerator();
    
    // Schedule regular duties
    this.scheduler.scheduleDaily('agentReview', this.conductAgentReviews.bind(this));
    this.scheduler.scheduleDaily('missionAlignment', this.ensureMissionAlignment.bind(this));
    this.scheduler.scheduleDaily('reportConsolidation', this.consolidateReports.bind(this));
    this.scheduler.scheduleWeekly('strategicPlanUpdate', this.updateStrategicPlan.bind(this));
    
    // Set up opportunity scanner
    this.scheduler.scheduleHourly('opportunityScan', this.scanForNewOpportunities.bind(this));
  }
  
  async processMessage(message) {
    switch (message.type) {
      case 'performance_report':
        return this.handlePerformanceReport(message);
        
      case 'market_update':
        return this.handleMarketUpdate(message);
        
      case 'sync_issues':
        return this.handleSyncIssues(message);
        
      case 'opportunity_alert':
        return this.handleOpportunityAlert(message);
        
      case 'agent_status_update':
        return this.handleAgentStatusUpdate(message);
        
      case 'user_request':
        return this.handleUserRequest(message);
        
      default:
        this.logger.warn(`Unknown message type received: ${message.type}`);
        return { error: 'Unknown message type' };
    }
  }
  
  async handlePerformanceReport(message) {
    const { reportId } = message;
    
    // Retrieve the report
    const report = await this.dataStore.getReport(reportId);
    
    if (!report) {
      return { error: 'Report not found' };
    }
    
    // Process the report
    this.logger.info(`Processing performance report: ${reportId}`);
    
    // Update metrics
    if (report.agentPerformance) {
      for (const agentPerf of report.agentPerformance) {
        await this.updateAgentMetrics(agentPerf.agentId, agentPerf.metrics);
      }
    }
    
    // Handle any anomalies
    if (report.anomalies && report.anomalies.length > 0) {
      await this.handleAnomalies(report.anomalies);
    }
    
    // Implement recommendations if they exist
    if (report.recommendations && report.recommendations.length > 0) {
      await this.implementRecommendations(report.recommendations);
    }
    
    return { success: true, reportId };
  }
  
  async handleMarketUpdate(message) {
    const { update } = message;
    
    this.logger.info('Processing market update', { update });
    
    // Update strategic plan if necessary
    const impactAssessment = await this.marketAnalytics.assessUpdateImpact(update);
    
    if (impactAssessment.significantImpact) {
      await this.updateStrategicPlan();
      
      // Notify agents of significant market changes
      await this.broadcastMessage({
        type: 'strategic_adjustment',
        changes: impactAssessment.changes,
        actionItems: impactAssessment.actionItems
      });
    }
    
    // Check for new opportunities
    if (update.opportunities && update.opportunities.length > 0) {
      for (const opportunity of update.opportunities) {
        await this.evaluateOpportunity(opportunity);
      }
    }
    
    return { success: true, impactAssessment };
  }
  
  async handleSyncIssues(message) {
    const { issues } = message;
    
    this.logger.info('Processing sync issues', { issueCount: issues.length });
    
    // Categorize issues by severity
    const categorizedIssues = {
      critical: issues.filter(i => i.severity === 'critical'),
      high: issues.filter(i => i.severity === 'high'),
      medium: issues.filter(i => i.severity === 'medium'),
      low: issues.filter(i => i.severity === 'low')
    };
    
    // Handle critical issues immediately
    for (const issue of categorizedIssues.critical) {
      await this.addressCriticalIssue(issue);
    }
    
    // Schedule resolution for other issues
    for (const issue of categorizedIssues.high) {
      this.scheduler.scheduleOnce(`issue_${issue.id}`, new Date(Date.now() + 1 * 60 * 60 * 1000), // 1 hour
        () => this.resolveIssue(issue));
    }
    
    for (const issue of categorizedIssues.medium) {
      this.scheduler.scheduleOnce(`issue_${issue.id}`, new Date(Date.now() + 4 * 60 * 60 * 1000), // 4 hours
        () => this.resolveIssue(issue));
    }
    
    // Log low severity issues but don't schedule immediate resolution
    for (const issue of categorizedIssues.low) {
      this.logger.info(`Low severity issue logged: ${issue.id}`);
    }
    
    return { 
      success: true, 
      issuesProcessed: issues.length,
      immediatelyAddressed: categorizedIssues.critical.length
    };
  }
  
  async handleOpportunityAlert(message) {
    const { opportunity } = message;
    
    this.logger.info('Processing opportunity alert', { opportunity });
    
    // Evaluate opportunity
    const evaluation = await this.evaluateOpportunity(opportunity);
    
    if (evaluation.viable) {
      // Add to opportunities list
      this.incomeOpportunities.push({
        ...opportunity,
        evaluation,
        detectedAt: new Date()
      });
      
      // Notify required agents
      for (const agentId of evaluation.requiredAgents) {
        await this.notifyAgent(agentId, {
          type: 'opportunity_assignment',
          opportunity,
          evaluation,
          instructions: evaluation.agentInstru