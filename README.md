# dual_hyperagent_system.py
# Complete unified system for HyperAgents v1.0
# Works with: https://github.com/facebookresearch/HyperAgents

import argparse
import json
import logging
import os
import re
import subprocess
import shutil
import time
from datetime import datetime
from concurrent.futures import ThreadPoolExecutor
from typing import Dict, List, Tuple, Any
from pathlib import Path

# Setup logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler("dual_hyperagent_system.log"),
        logging.StreamHandler()
    ]
)
logger = logging.getLogger(__name__)


# ============================================================================
# PART 1: LOG ANALYSIS FOR HYPERAGENTS
# ============================================================================

class HyperAgentsLogAnalyzer:
    """Analyzes logs from HyperAgents executions"""
    
    ERROR_PATTERNS = {
        "docker_error": r"(docker|Container|connection refused|Docker daemon)",
        "memory_error": r"(MemoryError|CUDA out of memory|RuntimeError|OOM)",
        "timeout_error": r"(timeout|timed out|TimeoutError)",
        "compilation_error": r"(SyntaxError|IndentationError|ImportError|ModuleNotFoundError)",
        "meta_agent_error": r"(meta_agent|run_meta_agent|model_patch|agent.*failed)",
        "task_agent_error": r"(task_agent.*error|TaskAgent|evaluation.*failed)",
        "evaluation_error": r"(evaluation failed|eval.*error|harness.*error|report\.json)",
        "git_error": r"(git error|fatal|merge conflict|git reset)",
        "domain_error": r"(domain.*error|balrog|genesis|polyglot.*error)",
    }
    
    def analyze_log(self, log_file: str) -> Tuple[List[str], List[str]]:
        """Analyzes generate.log file"""
        errors = []
        warnings = []
        
        if not os.path.exists(log_file):
            return errors, warnings
        
        try:
            with open(log_file, 'r') as f:
                content = f.read()
            
            # Check for errors
            for error_type, pattern in self.ERROR_PATTERNS.items():
                if re.search(pattern, content, re.IGNORECASE):
                    errors.append(error_type)
            
            # Check for warnings
            if 'warning' in content.lower():
                warnings.append("General warnings detected")
            if 'slow' in content.lower():
                warnings.append("Slow execution detected")
            if 'staged eval' in content.lower():
                warnings.append("Staged evaluation running")
            
        except Exception as e:
            logger.error(f"Error reading log file: {e}")
        
        return errors, warnings
    
    def analyze_metadata(self, metadata_file: str) -> Dict[str, Any]:
        """Analyzes metadata.json from gen_X directory"""
        metadata = {}
        
        if not os.path.exists(metadata_file):
            return metadata
        
        try:
            with open(metadata_file, 'r') as f:
                metadata = json.load(f)
        except Exception as e:
            logger.error(f"Error reading metadata: {e}")
        
        return metadata
    
    def extract_scores(self, eval_path: str) -> Dict[str, float]:
        """Extracts scores from report.json files"""
        scores = {}
        
        if not os.path.exists(eval_path):
            return scores
        
        try:
            # Look for report.json in domain_eval directories
            for root, dirs, files in os.walk(eval_path):
                for file in files:
                    if file == 'report.json':
                        report_path = os.path.join(root, file)
                        with open(report_path, 'r') as f:
                            report = json.load(f)
                            # Different domains have different score keys
                            for key in ['accuracy', 'score', 'mean_score', 'eval_score']:
                                if key in report:
                                    domain = os.path.basename(root)
                                    scores[domain] = float(report[key])
                                    break
        except Exception as e:
            logger.error(f"Error extracting scores: {e}")
        
        return scores


# ============================================================================
# PART 2: HEALTH CHECKING FOR HYPERAGENTS
# ============================================================================

class HyperAgentHealthChecker:
    """Checks health of a single HyperAgent instance"""
    
    def __init__(self, agent_id: str, output_dir: str):
        self.agent_id = agent_id
        self.output_dir = output_dir
        self.analyzer = HyperAgentsLogAnalyzer()
    
    def check_health(self) -> Dict[str, Any]:
        """Performs health check"""
        
        health = {
            "agent_id": self.agent_id,
            "timestamp": datetime.now().isoformat(),
            "errors": [],
            "warnings": [],
            "scores": {},
            "metadata": {},
            "generation_dirs": [],
            "health_score": 0,
        }
        
        # Find all generation directories
        gen_dirs = self._find_generation_dirs()
        if not gen_dirs:
            health["errors"].append("No generation directories found")
            return health
        
        health["generation_dirs"] = sorted(gen_dirs)
        latest_gen = gen_dirs[-1]  # Latest is last
        
        logger.info(f"Analyzing {self.agent_id}: Latest gen = {latest_gen}")
        
        # Analyze generate.log
        log_file = os.path.join(self.output_dir, latest_gen, "generate.log")
        errors, warnings = self.analyzer.analyze_log(log_file)
        health["errors"].extend(errors)
        health["warnings"].extend(warnings)
        
        # Analyze metadata.json
        metadata_file = os.path.join(self.output_dir, latest_gen, "metadata.json")
        metadata = self.analyzer.analyze_metadata(metadata_file)
        health["metadata"] = {
            "valid_parent": metadata.get("valid_parent", False),
            "run_eval": metadata.get("run_eval", False),
            "parent_agent_success": metadata.get("parent_agent_success", False),
            "run_full_eval": metadata.get("run_full_eval", False),
        }
        
        # Extract scores from all domain evals
        gen_path = os.path.join(self.output_dir, latest_gen)
        scores = self.analyzer.extract_scores(gen_path)
        health["scores"] = scores
        
        # Calculate health score (0-100)
        error_count = len(health["errors"])
        warning_count = len(health["warnings"])
        score_count = len(scores)
        
        health["health_score"] = max(0, 100 - (error_count * 15) - (warning_count * 5) + (score_count * 5))
        
        return health
    
    def _find_generation_dirs(self) -> List[str]:
        """Finds all gen_X directories"""
        if not os.path.exists(self.output_dir):
            return []
        
        gen_dirs = []
        try:
            for item in os.listdir(self.output_dir):
                if item.startswith('gen_') and os.path.isdir(os.path.join(self.output_dir, item)):
                    gen_dirs.append(item)
        except:
            pass
        
        return sorted(gen_dirs, key=lambda x: int(x.split('_')[1]) if x.split('_')[1].isdigit() else -1)


# ============================================================================
# PART 3: FIX APPLICATION
# ============================================================================

class HyperAgentFixApplier:
    """Applies fixes to HyperAgents"""
    
    def __init__(self):
        self.applied_fixes = []
        self.failed_fixes = []
    
    def cleanup_container_logs(self, output_dir: str) -> bool:
        """Cleans up docker container files and cache"""
        try:
            logger.info(f"🔧 Cleaning docker artifacts in {output_dir}")
            
            patterns = [
                '__pycache__',
                '.pytest_cache',
                '*.pyc',
                '.claude'
            ]
            
            for root, dirs, files in os.walk(output_dir):
                for pattern in patterns:
                    if pattern in dirs:
                        dir_path = os.path.join(root, pattern)
                        shutil.rmtree(dir_path, ignore_errors=True)
            
            self.applied_fixes.append({
                "name": "cleanup_cache",
                "type": "cleanup",
                "timestamp": datetime.now().isoformat()
            })
            logger.info(f"✅ Cleaned cache successfully")
            return True
        except Exception as e:
            self.failed_fixes.append({
                "name": "cleanup_cache",
                "error": str(e),
                "timestamp": datetime.now().isoformat()
            })
            return False
    
    def rebuild_docker_image(self, hyperagents_path: str) -> bool:
        """Rebuilds Docker image for HyperAgents"""
        try:
            logger.info(f"🔧 Rebuilding Docker image")
            
            cmd = ["docker", "build", "-t", "hyperagents", "--network=host", hyperagents_path]
            result = subprocess.run(cmd, capture_output=True, text=True, timeout=600)
            
            if result.returncode == 0:
                self.applied_fixes.append({
                    "name": "rebuild_docker",
                    "type": "command",
                    "command": " ".join(cmd),
                    "timestamp": datetime.now().isoformat()
                })
                logger.info(f"✅ Docker image rebuilt")
                return True
            else:
                self.failed_fixes.append({
                    "name": "rebuild_docker",
                    "error": result.stderr,
                    "timestamp": datetime.now().isoformat()
                })
                return False
        except Exception as e:
            self.failed_fixes.append({
                "name": "rebuild_docker",
                "error": str(e),
                "timestamp": datetime.now().isoformat()
            })
            return False


# ============================================================================
# PART 4: ORCHESTRATOR FOR HYPERAGENTS
# ============================================================================

class DualHyperAgentOrchestrator:
    """Orchestrates two HyperAgents running in parallel"""
    
    def __init__(self, hyperagents_path: str, domains: List[str], max_generation: int = 3):
        self.hyperagents_path = hyperagents_path
        self.domains = domains
        self.max_generation = max_generation
        self.timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
        
        # HyperAgents uses outputs/ directory
        self.output_dir = os.path.join(hyperagents_path, "outputs")
        self.agent1_run_id = f"agent1_{self.timestamp}"
        self.agent2_run_id = f"agent2_{self.timestamp}"
        
        self.processes = {}
        self.agent1_output_dir = None
        self.agent2_output_dir = None
    
    def start_agent(self, agent_id: str, run_id: str) -> bool:
        """Starts a single HyperAgent"""
        logger.info(f"🚀 Starting {agent_id}...")
        
        # HyperAgents command
        cmd = [
            "python",
            "generate_loop.py",
            "--domains",
            *self.domains,
            "--max_generation", str(self.max_generation),
            "--run_id", run_id
        ]
        
        try:
            process = subprocess.Popen(
                cmd,
                cwd=self.hyperagents_path,
                stdout=subprocess.PIPE,
                stderr=subprocess.PIPE,
                text=True
            )
            self.processes[agent_id] = process
            logger.info(f"✅ {agent_id} started (PID: {process.pid})")
            return True
        except Exception as e:
            logger.error(f"❌ Failed to start {agent_id}: {e}")
            return False
    
    def run_both_agents(self) -> bool:
        """Starts both agents in parallel"""
        
        logger.info("=" * 80)
        logger.info("DUAL HYPERAGENT ORCHESTRATION")
        logger.info("=" * 80)
        
        logger.info(f"\n📋 Configuration:")
        logger.info(f"   HyperAgents Path: {self.hyperagents_path}")
        logger.info(f"   Domains: {', '.join(self.domains)}")
        logger.info(f"   Max Generations: {self.max_generation}")
        logger.info(f"   Agent 1 Run ID: {self.agent1_run_id}")
        logger.info(f"   Agent 2 Run ID: {self.agent2_run_id}")
        
        # Start both agents in parallel
        with ThreadPoolExecutor(max_workers=2) as executor:
            future1 = executor.submit(self.start_agent, "agent_1", self.agent1_run_id)
            future2 = executor.submit(self.start_agent, "agent_2", self.agent2_run_id)
            
            success1 = future1.result()
            success2 = future2.result()
        
        if not (success1 and success2):
            logger.error("❌ Failed to start one or both agents")
            return False
        
        logger.info("\n✅ Both agents started! Running in parallel...")
        logger.info(f"   Agent 1 PID: {self.processes['agent_1'].pid}")
        logger.info(f"   Agent 2 PID: {self.processes['agent_2'].pid}")
        
        return True
    
    def monitor_agents(self, troubleshooter=None) -> Dict[str, bool]:
        """Monitors agents while they run"""
        
        logger.info("\n" + "=" * 80)
        logger.info("MONITORING DUAL AGENT EXECUTION")
        logger.info("=" * 80)
        
        check_count = 0
        while True:
            check_count += 1
            logger.info(f"\n[Check #{check_count}] {datetime.now().isoformat()}")
            
            # Check if agents are still running
            agent1_running = self.processes['agent_1'].poll() is None
            agent2_running = self.processes['agent_2'].poll() is None
            
            logger.info(f"  Agent 1: {'🟢 Running' if agent1_running else '🔴 Stopped'}")
            logger.info(f"  Agent 2: {'🟢 Running' if agent2_running else '🔴 Stopped'}")
            
            # Run troubleshooter periodically
            if troubleshooter and (check_count % 6 == 0):  # Every 30 minutes
                logger.info("\n  Running periodic health check...")
                self._run_periodic_check(troubleshooter)
            
            # Both finished
            if not agent1_running and not agent2_running:
                logger.info("\n✅ Both agents completed!")
                break
            
            # Wait before next check
            time.sleep(300)  # Check every 5 minutes
        
        return {
            "agent_1": self.processes['agent_1'].poll() == 0,
            "agent_2": self.processes['agent_2'].poll() == 0,
        }
    
    def _run_periodic_check(self, troubleshooter):
        """Runs periodic health check"""
        try:
            agent1_dir, agent2_dir = self.get_output_dirs()
            if agent1_dir and agent2_dir:
                report = troubleshooter.run_full_workflow(enable_autofix=False)
                logger.info(f"  Agent 1 Health: {report['agent1']['health_score']}/100")
                logger.info(f"  Agent 2 Health: {report['agent2']['health_score']}/100")
        except Exception as e:
            logger.warning(f"  Health check failed: {e}")
    
    def get_output_dirs(self) -> Tuple[str, str]:
        """Returns output directories for both agents"""
        
        # Look for generate_XXXXX directories created by HyperAgents
        agent1_dir = os.path.join(self.output_dir, f"generate_{self.agent1_run_id}")
        agent2_dir = os.path.join(self.output_dir, f"generate_{self.agent2_run_id}")
        
        self.agent1_output_dir = agent1_dir if os.path.exists(agent1_dir) else None
        self.agent2_output_dir = agent2_dir if os.path.exists(agent2_dir) else None
        
        return self.agent1_output_dir, self.agent2_output_dir
    
    def wait_for_completion(self, troubleshooter=None) -> Dict[str, bool]:
        """Waits for both agents to complete"""
        return self.monitor_agents(troubleshooter=troubleshooter)


# ============================================================================
# PART 5: TROUBLESHOOTER & AUTOFIX
# ============================================================================

class DualHyperAgentTroubleshooter:
    """Troubleshoots and autofixes dual HyperAgent instances"""
    
    def __init__(self, hyperagents_path: str, agent1_output: str, agent2_output: str):
        self.hyperagents_path = hyperagents_path
        self.agent1_checker = HyperAgentHealthChecker("agent_1", agent1_output)
        self.agent2_checker = HyperAgentHealthChecker("agent_2", agent2_output)
        self.fix_applier = HyperAgentFixApplier()
    
    def run_full_workflow(self, enable_autofix: bool = False) -> Dict[str, Any]:
        """Runs complete troubleshooting workflow"""
        
        logger.info("\n" + "=" * 80)
        logger.info("TROUBLESHOOTING & DIAGNOSTICS")
        logger.info("=" * 80)
        
        report = {
            "timestamp": datetime.now().isoformat(),
            "mode": "troubleshoot_and_autofix" if enable_autofix else "troubleshoot_only",
        }
        
        # Check both agents
        logger.info("\nChecking Agent 1...")
        agent1_health = self.agent1_checker.check_health()
        logger.info(f"Agent 1 Health: {agent1_health['health_score']}/100")
        logger.info(f"Agent 1 Errors: {len(agent1_health['errors'])}")
        
        logger.info("\nChecking Agent 2...")
        agent2_health = self.agent2_checker.check_health()
        logger.info(f"Agent 2 Health: {agent2_health['health_score']}/100")
        logger.info(f"Agent 2 Errors: {len(agent2_health['errors'])}")
        
        # Comparison
        comparison = {
            "agent1_health": agent1_health['health_score'],
            "agent2_health": agent2_health['health_score'],
            "better_agent": "agent_1" if agent1_health['health_score'] > agent2_health['health_score'] else "agent_2",
            "difference": abs(agent1_health['health_score'] - agent2_health['health_score']),
        }
        
        report["agent1"] = agent1_health
        report["agent2"] = agent2_health
        report["comparison"] = comparison
        
        # AutoFix if enabled
        if enable_autofix:
            logger.info("\n" + "=" * 80)
            logger.info("AUTOFIX APPLICATION")
            logger.info("=" * 80)
            
            issues = self._extract_issues(agent1_health, agent2_health)
            logger.info(f"\nFound {len(issues)} issues")
            
            if issues:
                self._apply_fixes(issues)
            
            report["autofix"] = {
                "issues_found": len(issues),
                "fixes_applied": len(self.fix_applier.applied_fixes),
                "fixes_failed": len(self.fix_applier.failed_fixes),
            }
        
        # Summary
        logger.info("\n" + "=" * 80)
        logger.info("SUMMARY")
        logger.info("=" * 80)
        logger.info(f"Agent 1 Health: {agent1_health['health_score']}/100")
        logger.info(f"Agent 2 Health: {agent2_health['health_score']}/100")
        logger.info(f"Better Agent: {comparison['better_agent']}")
        
        recommendations = self._get_recommendations(agent1_health, agent2_health)
        for rec in recommendations:
            logger.info(f"  • {rec}")
        
        report["recommendations"] = recommendations
        
        return report
    
    def _extract_issues(self, health1: Dict, health2: Dict) -> List[Dict]:
        """Extracts issues"""
        issues = []
        
        for error in health1['errors']:
            issues.append({
                "agent": "agent_1",
                "error": error,
                "severity": "high"
            })
        
        for error in health2['errors']:
            issues.append({
                "agent": "agent_2",
                "error": error,
                "severity": "high"
            })
        
        return issues
    
    def _apply_fixes(self, issues: List[Dict]):
        """Applies fixes"""
        
        # Group by agent
        agent1_issues = [i for i in issues if i['agent'] == 'agent_1']
        agent2_issues = [i for i in issues if i['agent'] == 'agent_2']
        
        if agent1_issues or agent2_issues:
            self.fix_applier.cleanup_container_logs(self.hyperagents_path)
            self.fix_applier.rebuild_docker_image(self.hyperagents_path)
    
    def _get_recommendations(self, health1: Dict, health2: Dict) -> List[str]:
        """Gets recommendations"""
        recs = []
        
        if len(health1['errors']) > len(health2['errors']):
            recs.append("Agent 1 has more errors - check generate.log")
        
        if len(health2['errors']) > len(health1['errors']):
            recs.append("Agent 2 has more errors - check generate.log")
        
        if health1['health_score'] < 70:
            recs.append("Agent 1 health is critical - may need intervention")
        
        if health2['health_score'] < 70:
            recs.append("Agent 2 health is critical - may need intervention")
        
        if not recs:
            recs.append("Both agents are healthy!")
        
        return recs


# ============================================================================
# MAIN
# ============================================================================

def main():
    parser = argparse.ArgumentParser(
        description="Dual HyperAgent Orchestration, Monitoring & Troubleshooting System"
    )
    
    parser.add_argument(
        "--hyperagents_path",
        type=str,
        required=True,
        help="Path to HyperAgents repository"
    )
    parser.add_argument(
        "--domains",
        type=str,
        nargs="+",
        default=["search_arena"],
        help="Domains (e.g., search_arena paper_review)"
    )
    parser.add_argument(
        "--max_generation",
        type=int,
        default=3,
        help="Maximum generations"
    )
    parser.add_argument(
        "--enable_troubleshooter",
        action="store_true",
        help="Enable troubleshooting"
    )
    parser.add_argument(
        "--enable_autofix",
        action="store_true",
        help="Enable autofix"
    )
    parser.add_argument(
        "--output",
        type=str,
        default="dual_hyperagent_report.json",
        help="Output report path"
    )
    
    args = parser.parse_args()
    
    logger.info("\n" + "=" * 80)
    logger.info("DUAL HYPERAGENT SYSTEM v1.0")
    logger.info("=" * 80)
    logger.info("✅ Orchestrates two HyperAgents side by side")
    logger.info("✅ Monitors health during execution")
    logger.info("✅ Troubleshoots and detects issues")
    logger.info("✅ Automatically applies fixes")
    logger.info("=" * 80)
    
    # Create orchestrator
    orchestrator = DualHyperAgentOrchestrator(
        args.hyperagents_path,
        args.domains,
        args.max_generation
    )
    
    # Create troubleshooter
    troubleshooter = None
    if args.enable_troubleshooter:
        agent1_dir, agent2_dir = orchestrator.get_output_dirs()
        if agent1_dir and agent2_dir:
            troubleshooter = DualHyperAgentTroubleshooter(
                args.hyperagents_path,
                agent1_dir,
                agent2_dir
            )
    
    try:
        # Start both agents
        if not orchestrator.run_both_agents():
            logger.error("❌ Failed to start agents")
            return
        
        # Wait for completion
        logger.info("\n⏳ Waiting for agents to complete...")
        logger.info("   (This may take several hours)")
        
        results = orchestrator.wait_for_completion(troubleshooter=troubleshooter)
        
        # Final report
        final_report = {
            "timestamp": datetime.now().isoformat(),
            "execution_results": results,
        }
        
        if args.enable_troubleshooter:
            agent1_dir, agent2_dir = orchestrator.get_output_dirs()
            
            if agent1_dir and agent2_dir:
                logger.info("\n" + "=" * 80)
                logger.info("FINAL TROUBLESHOOTING")
                logger.info("=" * 80)
                
                troubleshoot_report = troubleshooter.run_full_workflow(
                    enable_autofix=args.enable_autofix
                )
                final_report["troubleshooting"] = troubleshoot_report
        
        # Save report
        with open(args.output, 'w') as f:
            json.dump(final_report, f, indent=2)
        
        logger.info("\n" + "=" * 80)
        logger.info("✅ EXECUTION COMPLETE")
        logger.info("=" * 80)
        logger.info(f"Agent 1: {'✅ Success' if results['agent_1'] else '❌ Failed'}")
        logger.info(f"Agent 2: {'✅ Success' if results['agent_2'] else '❌ Failed'}")
        logger.info(f"Report: {args.output}")
        logger.info("=" * 80 + "\n")
    
    except KeyboardInterrupt:
        logger.warning("\n⚠️  Interrupted by user")
        for agent_id, process in orchestrator.processes.items():
            if process.poll() is None:
                logger.info(f"Terminating {agent_id}...")
                process.terminate()
                try:
                    process.wait(timeout=10)
                except:
                    process.kill()
        logger.info("✅ Agents terminated")
    
    except Exception as e:
        logger.error(f"❌ Error: {e}", exc_info=True)


if __name__ == "__main__":
    main()
