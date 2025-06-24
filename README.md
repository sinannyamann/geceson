const express = require('express');
const EventEmitter = require('events');
const axios = require('axios');

class GecexCore extends EventEmitter {
  constructor() {
    super();
    this.app = express();
    this.plugins = new Map();
    this.services = new Map();
    this.middleware = [];
    this.config = {
      port: process.env.PORT || process.env.GECEX_PORT || 4000, // Use Railway's PORT
      environment: process.env.NODE_ENV || 'development',
      logLevel: process.env.LOG_LEVEL || 'info',
      enableMetrics: true,
      enableHealthCheck: true
    };
    
    this.metrics = {
      requests: 0,
      errors: 0,
      activeConnections: 0,
      pluginCalls: {},
      startTime: Date.now()
    };
    
    this.setupCore();
  }

  setupCore() {
    this.app.use(express.json({ limit: '10mb' }));
    this.app.use(express.urlencoded({ extended: true }));
    
    // Request tracking middleware
    this.app.use((req, res, next) => {
      this.metrics.requests++;
      this.metrics.activeConnections++;
      
      req.gecexId = `req_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
      req.startTime = Date.now();
      
      res.on('finish', () => {
        this.metrics.activeConnections--;
        const duration = Date.now() - req.startTime;
        this.emit('request:completed', {
          id: req.gecexId,
          method: req.method,
          path: req.path,
          statusCode: res.statusCode,
          duration
        });
      });
      
      next();
    });
    
    this.setupCoreRoutes();
  }

  setupCoreRoutes() {
    // Core platform endpoints
    this.app.get('/gecex/health', (req, res) => {
      const uptime = Date.now() - this.metrics.startTime;
      const health = {
        status: 'healthy',
        platform: 'GecexCore',
        version: '1.0.0',
        uptime: Math.floor(uptime / 1000),
        memory: {
          used: Math.round(process.memoryUsage().heapUsed / 1024 / 1024),
          total: Math.round(process.memoryUsage().heapTotal / 1024 / 1024)
        },
        plugins: {
          total: this.plugins.size,
          active: Array.from(this.plugins.entries()).filter(([name, plugin]) => plugin.status === 'active').length
        },
        services: {
          total: this.services.size,
          healthy: Array.from(this.services.values()).filter(service => service.health === 'healthy').length
        },
        metrics: this.metrics
      };
      
      res.json(health);
    });

    this.app.get('/gecex/plugins', (req, res) => {
      const pluginList = Array.from(this.plugins.entries()).map(([name, plugin]) => ({
        name,
        status: plugin.status,
        version: plugin.version,
        description: plugin.description,
        endpoints: plugin.endpoints || [],
        lastActivity: plugin.lastActivity
      }));
      
      res.json({ plugins: pluginList });
    });

    this.app.get('/gecex/services', (req, res) => {
      const serviceList = Array.from(this.services.entries()).map(([name, service]) => ({
        name,
        url: service.url,
        health: service.health,
        lastCheck: service.lastCheck,
        responseTime: service.responseTime
      }));
      
      res.json({ services: serviceList });
    });

    // Unified API endpoint - routes to appropriate plugins
    this.app.all('/api/*', async (req, res) => {
      const path = req.path.replace('/api/', '');
      const segments = path.split('/');
      const pluginName = segments[0];
      
      try {
        const result = await this.callPlugin(pluginName, req.method, path, req.body, req.query, req.headers);
        res.status(result.statusCode || 200).json(result.data);
      } catch (error) {
        this.metrics.errors++;
        this.emit('error', { plugin: pluginName, error: error.message, request: req.gecexId });
        
        res.status(500).json({
          error: 'Plugin execution failed',
          plugin: pluginName,
          message: error.message,
          requestId: req.gecexId
        });
      }
    });

    // Advanced chat endpoint - orchestrates multiple plugins
    this.app.post('/gecex/chat', async (req, res) => {
      const { message, username, context } = req.body;
      
      if (!message) {
        return res.status(400).json({ error: 'Message is required' });
      }
      
      try {
        const orchestrationResult = await this.orchestrateChat({
          message,
          username: username || 'anonymous',
          context: context || {},
          requestId: req.gecexId
        });
        
        res.json(orchestrationResult);
      } catch (error) {
        this.metrics.errors++;
        res.status(500).json({
          error: 'Chat orchestration failed',
          message: error.message,
          requestId: req.gecexId
        });
      }
    });

    // Plugin management endpoints
    this.app.post('/gecex/plugins/:name/enable', (req, res) => {
      const { name } = req.params;
      const plugin = this.plugins.get(name);
      
      if (!plugin) {
        return res.status(404).json({ error: 'Plugin not found' });
      }
      
      plugin.status = 'active';
      plugin.lastActivity = new Date().toISOString();
      
      this.emit('plugin:enabled', { name, plugin });
      res.json({ success: true, plugin: name, status: 'active' });
    });

    this.app.post('/gecex/plugins/:name/disable', (req, res) => {
      const { name } = req.params;
      const plugin = this.plugins.get(name);
      
      if (!plugin) {
        return res.status(404).json({ error: 'Plugin not found' });
      }
      
      plugin.status = 'inactive';
      plugin.lastActivity = new Date().toISOString();
      
      this.emit('plugin:disabled', { name, plugin });
      res.json({ success: true, plugin: name, status: 'inactive' });
    });
  }

  // Plugin Registration System
  registerPlugin(name, pluginConfig) {
    const plugin = {
      name,
      version: pluginConfig.version || '1.0.0',
      description: pluginConfig.description || '',
      endpoints: pluginConfig.endpoints || [],
      handler: pluginConfig.handler,
      status: 'active',
      registeredAt: new Date().toISOString(),
      lastActivity: new Date().toISOString(),
      config: pluginConfig.config || {}
    };
    
    this.plugins.set(name, plugin);
    this.metrics.pluginCalls[name] = 0;
    
    // Auto-register plugin routes if provided
    if (pluginConfig.routes) {
      Object.entries(pluginConfig.routes).forEach(([route, handler]) => {
        const fullRoute = `/api/${name}${route}`;
        this.app.all(fullRoute, async (req, res) => {
          try {
            const result = await handler(req, res, this);
            if (!res.headersSent) {
              res.json(result);
            }
          } catch (error) {
            if (!res.headersSent) {
              res.status(500).json({ error: error.message });
            }
          }
        });
      });
    }
    
    this.emit('plugin:registered', { name, plugin });
    this.log('info', `Plugin registered: ${name} v${plugin.version}`);
    
    return plugin;
  }

  // Service Registration (External Microservices)
  registerService(name, serviceConfig) {
    const service = {
      name,
      url: serviceConfig.url,
      health: 'unknown',
      timeout: serviceConfig.timeout || 5000,
      retries: serviceConfig.retries || 3,
      lastCheck: null,
      responseTime: null,
      registeredAt: new Date().toISOString()
    };
    
    this.services.set(name, service);
    this.emit('service:registered', { name, service });
    this.log('info', `Service registered: ${name} at ${service.url}`);
    
    // Start health monitoring
    this.startServiceHealthCheck(name);
    
    return service;
  }

  // Plugin Execution
  async callPlugin(pluginName, method, path, body, query, headers) {
    const plugin = this.plugins.get(pluginName);
    
    if (!plugin) {
      throw new Error(`Plugin not found: ${pluginName}`);
    }
    
    if (plugin.status !== 'active') {
      throw new Error(`Plugin not active: ${pluginName}`);
    }
    
    this.metrics.pluginCalls[pluginName]++;
    plugin.lastActivity = new Date().toISOString();
    
    const context = {
      method,
      path,
      body,
      query,
      headers,
      gecex: this,
      plugin: plugin.name
    };
    
    try {
      const result = await plugin.handler(context);
      this.emit('plugin:called', { plugin: pluginName, success: true });
      return result;
    } catch (error) {
      this.emit('plugin:called', { plugin: pluginName, success: false, error: error.message });
      throw error;
    }
  }

  // Service Health Monitoring
  async startServiceHealthCheck(serviceName) {
    const service = this.services.get(serviceName);
    if (!service) return;
    
    const checkHealth = async () => {
      const startTime = Date.now();
      try {
        const response = await axios.get(`${service.url}/health`, {
          timeout: service.timeout
        });
        
        service.health = response.status === 200 ? 'healthy' : 'unhealthy';
        service.responseTime = Date.now() - startTime;
        service.lastCheck = new Date().toISOString();
        
      } catch (error) {
        service.health = 'unhealthy';
        service.responseTime = Date.now() - startTime;
        service.lastCheck = new Date().toISOString();
      }
    };
    
    // Initial check
    await checkHealth();
    
    // Periodic checks every 30 seconds
    setInterval(checkHealth, 30000);
  }

  // Advanced Chat Orchestration
  async orchestrateChat(chatRequest) {
    const { message, username, context, requestId } = chatRequest;
    const orchestration = {
      requestId,
      steps: [],
      result: {},
      timing: {
        start: Date.now(),
        end: null,
        duration: null
      }
    };
    
    try {
      // Step 1: User Character Analysis
      if (this.plugins.has('character')) {
        orchestration.steps.push('character_analysis');
        const userAnalysis = await this.callPlugin('character', 'POST', 'character/analyze', {
          username,
          message,
          context
        });
        orchestration.result.user = userAnalysis.data;
      }
      
      // Step 2: AI Response Generation
      if (this.plugins.has('ai')) {
        orchestration.steps.push('ai_generation');
        const aiResponse = await this.callPlugin('ai', 'POST', 'ai/chat', {
          message,
          userContext: orchestration.result.user,
          context
        });
        orchestration.result.response = aiResponse.data;
      }
      
      // Step 3: Logging & Analytics
      if (this.plugins.has('analytics')) {
        orchestration.steps.push('analytics_logging');
        await this.callPlugin('analytics', 'POST', 'analytics/record', {
          event: 'chat_interaction',
          username,
          message,
          response: orchestration.result.response,
          requestId
        });
      }
      
      // Step 4: Response Enhancement
      if (this.plugins.has('enhancement')) {
        orchestration.steps.push('response_enhancement');
        const enhanced = await this.callPlugin('enhancement', 'POST', 'enhancement/process', {
          response: orchestration.result.response,
          userContext: orchestration.result.user
        });
        orchestration.result.response = enhanced.data;
      }
      
    } catch (error) {
      orchestration.error = error.message;
    }
    
    orchestration.timing.end = Date.now();
    orchestration.timing.duration = orchestration.timing.end - orchestration.timing.start;
    
    this.emit('chat:orchestrated', orchestration);
    
    return {
      response: orchestration.result.response?.response || 'Orchestration failed',
      sender: 'GecexCore',
      timestamp: new Date().toISOString(),
      orchestration: {
        requestId,
        steps: orchestration.steps,
        duration: orchestration.timing.duration,
        success: !orchestration.error
      },
      user: orchestration.result.user
    };
  }

  // Logging System
  log(level, message, data = {}) {
    const logEntry = {
      timestamp: new Date().toISOString(),
      level,
      message,
      data,
      platform: 'GecexCore'
    };
    
    console.log(`[${logEntry.timestamp}] [${level.toUpperCase()}] ${message}`, data);
    this.emit('log', logEntry);
  }

  // Platform Startup
  async start() {
    return new Promise((resolve) => {
      this.app.listen(this.config.port, () => {
        this.log('info', `GecexCore Platform started on port ${this.config.port}`);
        this.log('info', `Environment: ${this.config.environment}`);
        this.log('info', `Health Check: http://localhost:${this.config.port}/gecex/health`);
        this.log('info', `Plugin API: http://localhost:${this.config.port}/api/*`);
        this.log('info', `Advanced Chat: http://localhost:${this.config.port}/gecex/chat`);
        
        this.emit('platform:started');
        resolve();
      });
    });
  }

  // Graceful Shutdown
  async shutdown() {
    this.log('info', 'GecexCore Platform shutting down...');
    
    // Notify all plugins
    this.emit('platform:shutdown');
    
    // Close server
    return new Promise((resolve) => {
      this.app.close(() => {
        this.log('info', 'GecexCore Platform stopped');
        resolve();
      });
    });
  }
}

module.exports = GecexCore;

// Start the server
const core = new GecexCore();
core.start();
