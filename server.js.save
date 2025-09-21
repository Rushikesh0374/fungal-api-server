const express = require('express');
const fs = require('fs');
const { exec } = require('child_process');
const app = express();
const PORT = 4000;

// Middleware to parse JSON bodies
app.use(express.json());

// Helper function to format process data consistently
const formatProcessData = (processArray) => {
  return processArray.map(proc => ({
    id: proc.pm_id,
    name: proc.name,
    status: proc.pm2_env.status,
    pid: proc.pid,
    cpu: proc.monit.cpu,
    memory: proc.monit.memory,
    uptime: proc.pm2_env.pm_uptime,
    restarts: proc.pm2_env.restart_time,
    script: proc.pm2_env.pm_exec_path,
    args: proc.pm2_env.args || [],
    created_at: proc.pm2_env.created_at,
    unstable_restarts: proc.pm2_env.unstable_restarts || 0
  }));
};

// Helper function to execute PM2 commands with consistent JSON response
const executePM2Command = (command, res, successMessage = 'Command executed successfully') => {
  exec(command, (error, stdout, stderr) => {
    if (error) {
      return res.status(500).json({
        success: false,
        error: 'PM2 command failed',
        message: error.message,
        code: error.code || 'UNKNOWN_ERROR',
        timestamp: new Date().toISOString()
      });
    }
    
    try {
      // Handle jlist commands - return formatted process data
      if (command.includes('jlist')) {
        const jsonData = JSON.parse(stdout);
        return res.json({
          success: true,
          data: formatProcessData(jsonData),
          count: jsonData.length,
          timestamp: new Date().toISOString()
        });
      } 
      // Handle other commands
      else {
        return res.json({
          success: true,
          message: successMessage,
          output: stdout.trim(),
          timestamp: new Date().toISOString()
        });
      }
    } catch (parseError) {
      return res.json({
        success: true,
        message: successMessage,
        output: stdout.trim(),
        timestamp: new Date().toISOString()
      });
    }
  });
};

// JSON API endpoint
app.get('/api/json', (req, res) => {
  fs.readFile('data.json', 'utf8', (err, data) => {
    if (err) {
      return res.status(500).json({
        success: false,
        error: 'Failed to read data.json',
        message: err.message
      });
    }
    
    try {
      const jsonData = JSON.parse(data);
      res.json({
        success: true,
        data: jsonData
      });
    } catch (parseErr) {
      res.status(500).json({
        success: false,
        error: 'Invalid JSON format in data.json',
        message: parseErr.message
      });
    }
  });
});

// PM2 Management APIs

// Get all PM2 processes
app.get('/api/pm2/processes', (req, res) => {
  executePM2Command('pm2 jlist', res);
});

// Get specific process info
app.get('/api/pm2/processes/:id', (req, res) => {
  const processId = req.params.id;
  
  exec('pm2 jlist', (error, stdout, stderr) => {
    if (error) {
      return res.status(500).json({
        success: false,
        error: 'Failed to fetch process info',
        message: error.message,
        code: 'PM2_ERROR',
        timestamp: new Date().toISOString()
      });
    }
    
    try {
      const processes = JSON.parse(stdout);
      const process = processes.find(p => 
        p.pm_id.toString() === processId || p.name === processId
      );
      
      if (!process) {
        return res.status(404).json({
          success: false,
          error: 'Process not found',
          message: `No process found with ID or name: ${processId}`,
          code: 'PROCESS_NOT_FOUND',
          timestamp: new Date().toISOString()
        });
      }
      
      res.json({
        success: true,
        data: formatProcessData([process])[0],
        timestamp: new Date().toISOString()
      });
      
    } catch (parseError) {
      res.status(500).json({
        success: false,
        error: 'Failed to parse process data',
        message: parseError.message,
        code: 'PARSE_ERROR',
        timestamp: new Date().toISOString()
      });
    }
  });
});

// Start a process
app.post('/api/pm2/start', (req, res) => {
  const { name, script, args, instances } = req.body;
  
  if (!script) {
    return res.status(400).json({
      success: false,
      error: 'Validation error',
      message: 'Script path is required',
      code: 'MISSING_SCRIPT',
      timestamp: new Date().toISOString()
    });
  }
  
  let command = `pm2 start ${script}`;
  if (name) command += ` --name "${name}"`;
  if (args) command += ` -- ${args}`;
  if (instances) command += ` -i ${instances}`;
    
  executePM2Command(command, res, `Process started successfully`);
});

// Restart process(es)
app.post('/api/pm2/restart', (req, res) => {
  const { id } = req.body;
  
  if (!id) {
    return res.status(400).json({
      success: false,
      error: 'Validation error',
      message: 'Process ID or name is required',
      code: 'MISSING_PROCESS_ID',
      timestamp: new Date().toISOString()
    });
  }
  
  executePM2Command(`pm2 restart ${id}`, res, `Process '${id}' restarted successfully`);
});

// Restart all processes
app.post('/api/pm2/restart-all', (req, res) => {
  executePM2Command('pm2 restart all', res, 'All processes restarted successfully');
});

// Stop process(es)
app.post('/api/pm2/stop', (req, res) => {
  const { id } = req.body;
  
  if (!id) {
    return res.status(400).json({
      success: false,
      error: 'Validation error',
      message: 'Process ID or name is required',
      code: 'MISSING_PROCESS_ID',
      timestamp: new Date().toISOString()
    });
  }
  
  executePM2Command(`pm2 stop ${id}`, res, `Process '${id}' stopped successfully`);
});

// Stop all processes
app.post('/api/pm2/stop-all', (req, res) => {
  executePM2Command('pm2 stop all', res, 'All processes stopped successfully');
});

// Delete process(es)
app.delete('/api/pm2/delete/:id', (req, res) => {
  const processId = req.params.id;
  executePM2Command(`pm2 delete ${processId}`, res, `Process '${processId}' deleted successfully`);
});

// Delete all processes
app.delete('/api/pm2/delete-all', (req, res) => {
  executePM2Command('pm2 delete all', res, 'All processes deleted successfully');
});

// Reload process(es) - for 0 downtime reload
app.post('/api/pm2/reload', (req, res) => {
  const { id } = req.body;
  
  if (!id) {
    return res.status(400).json({
      success: false,
      error: 'Validation error',
      message: 'Process ID or name is required',
      code: 'MISSING_PROCESS_ID',
      timestamp: new Date().toISOString()
    });
  }
  
  executePM2Command(`pm2 reload ${id}`, res, `Process '${id}' reloaded successfully`);
});

// Get PM2 logs for a specific process
app.get('/api/pm2/logs/:id', (req, res) => {
  const processId = req.params.id;
  const lines = req.query.lines || 50;
  
  exec(`pm2 logs ${processId} --lines ${lines} --nostream`, (error, stdout, stderr) => {
    if (error) {
      return res.status(500).json({
        success: false,
        error: 'Failed to fetch logs',
        message: error.message,
        code: 'LOGS_ERROR',
        timestamp: new Date().toISOString()
      });
    }
    
    // Parse logs into structured format
    const logLines = stdout.split('\n')
      .filter(line => line.trim())
      .map(line => {
        const match = line.match(/^(\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}): (.+)$/);
        return {
          timestamp: match ? match[1] : new Date().toISOString(),
          message: match ? match[2] : line,
          raw: line
        };
      });
    
    res.json({
      success: true,
      data: {
        processId: processId,
        lines: parseInt(lines),
        logs: logLines,
        totalLines: logLines.length
      },
      timestamp: new Date().toISOString()
    });
  });
});

// Flush logs
app.post('/api/pm2/flush-logs', (req, res) => {
  executePM2Command('pm2 flush', res, 'All logs flushed successfully');
});

// Get PM2 system info
app.get('/api/pm2/info', (req, res) => {
  exec('pm2 jlist', (error, stdout, stderr) => {
    if (error) {
      return res.status(500).json({
        success: false,
        error: 'Failed to fetch PM2 info',
        message: error.message,
        code: 'INFO_ERROR',
        timestamp: new Date().toISOString()
      });
    }
    
    try {
      const processes = JSON.parse(stdout);
      const totalProcesses = processes.length;
      const runningProcesses = processes.filter(p => p.pm2_env.status === 'online').length;
      const stoppedProcesses = processes.filter(p => p.pm2_env.status === 'stopped').length;
      const erroredProcesses = processes.filter(p => p.pm2_env.status === 'errored').length;
      
      res.json({
        success: true,
        data: {
          totalProcesses,
          runningProcesses,
          stoppedProcesses,
          erroredProcesses,
          systemInfo: {
            nodeVersion: process.version,
            platform: process.platform,
            arch: process.arch,
            uptime: process.uptime()
          }
        },
        timestamp: new Date().toISOString()
      });
    } catch (parseError) {
      res.status(500).json({
        success: false,
        error: 'Failed to parse PM2 data',
        message: parseError.message,
        code: 'PARSE_ERROR',
        timestamp: new Date().toISOString()
      });
    }
  });
});

app.listen(PORT, () => {
  console.log(`🚀 Server running at http://localhost:${PORT}`);
  console.log(`📋 JSON API: http://localhost:${PORT}/api/json`);
  console.log(`⚙️  PM2 APIs:`);
  console.log(`   GET    /api/pm2/processes - List all processes`);
  console.log(`   GET    /api/pm2/processes/:id - Get specific process`);
  console.log(`   POST   /api/pm2/start - Start process`);
  console.log(`   POST   /api/pm2/restart - Restart process`);
  console.log(`   POST   /api/pm2/stop - Stop process`);
  console.log(`   DELETE /api/pm2/delete/:id - Delete process`);
  console.log(`   GET    /api/pm2/logs/:id - Get process logs`);
});
