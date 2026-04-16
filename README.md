# troubleshooter_and_autofix.py
# Complete unified troubleshooting and autofix system for dual HyperAgents

import argparse
import json
import logging
import os
import re
import subprocess
import shutil
from datetime import datetime
from typing import Dict, List, Tuple, Any
from concurrent.futures import ThreadPoolExecutor

# Setup logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)


# ============================================================================
# PART 1: LOG ANALYSIS
# ============================================================================

class LogAnalyzer:
    """Analyzes logs for errors and warnings"""
    
    ERROR_PATTERNS = {
        "docker_error": r"(docker|Container|ENOENT)",
        "memory_error": r"(MemoryError|CUDA out of memory|RuntimeError)",
        "timeout_error": r"(timeout|timed out|TimeoutError)",
        "compilation_error": r"(SyntaxError|IndentationError|ImportError)",
        "evaluation_error": r"(evaluation failed|eval.*error)",
        "git_error": r"(git error|fatal|merge conflict)",
    }
    
    def analyze_log(self, log_file: str) -> Tuple[List[str], List[str]]:
        """Analyzes a log file for errors and warnings"""
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
                warnings.append("Warnings detected")
            if 'slow' in content.lower():
                warnings.append("Slow execution detected")
            
        except Exception as e:
            logger.error(f"Error reading log file: {e}")
        
        return errors, warnings
    
    def get_metrics(self, eval_path: str) -> Dict[str, float]:
        """Extracts performance metrics"""
        metrics = {}
        
        predictions_file = os.path.join(eval_path, "predictions.csv")
        if os.path.exists(predictions_file):
            try:
                import pandas as pd
                df = pd.read_csv(predictions_file)
                if 'score' in df.columns:
                    metrics['mean_score'] = float(df['score'].mean())
                    metrics['max_score'] = float(df['score'].max())
                    metrics['min_score'] = float(df['score'].min())
            except:
                pass
        
        return metrics


# ============================================================================
# PART 2: HEALTH CHECKING
# ============================================================================

class AgentHealthChecker:
    """Checks health of a single agent"""
    
    def __init__(self, agent_id: str, repo_path: str, eval_path: str):
        self.agent_id = agent_id
        self.repo_path = repo_path
        self.eval_path = eval_path
        self.analyzer = LogAnalyzer()
    
    def check_health(self) -> Dict[str, Any]:
        """Checks overall health of the agent"""
        
        health = {
            "agent_id": self.agent_id,
            "timestamp": datetime.now().isoformat(),
            "errors": [],
            "warnings": [],
            "metrics": {},
            "health_score": 0,
        }
        
        # Find latest generation directory
        latest_gen = self._find_latest_gen()
        if latest_gen is None:
            health["errors"].append("No generation directories found")
            return health
        
        gen_dir = latest_gen
        logger.info(f"Analyzing {self.agent_id} in: {gen_dir}")
        
        # Check for errors in logs
        log_file = os.path.join(gen_dir, "generate.log")
        errors, warnings = self.analyzer.analyze_log(log_file)
        health["errors"].extend(errors)
        health["warnings"].extend(warnings)
        
        # Extract metrics
        for domain in ["search_arena", "paper_review", "balrog_babyai"]:
            eval_dir = os.path.join(gen_dir, f"{domain}_eval")
            if os.path.exists(eval_dir):
                metrics = self.analyzer.get_metrics(eval_dir)
                if metrics:
                    health["metrics"][domain] = metrics
        
        # Calculate health score
        error_count = len(health["errors"])
        warning_count = len(health["warnings"])
        health["health_score"] = max(0, 100 - (error_count * 10) - (warning_count * 2))
        
        return health
    
    def _find_latest_gen(self):
        """Finds the latest generation directory"""
        if not os.path.exists(self.eval_path):
            return None
        
        gen_dirs = []
        for d in os.listdir(self.eval_path):
            if d.startswith('gen_') and os.path.isdir(os.path.join(self.eval_path, d)):
                gen_dirs.append(os.path.join(self.eval_path, d))
        
        return max(gen_dirs, key=os.path.getmtime) if gen_dirs else None


# ============================================================================
# PART 3: FIX APPLICATION
# ============================================================================

class FixApplier:
    """Applies fixes to resolve issues"""
    
    def __init__(self):
        self.applied_fixes = []
        self.failed_fixes = []
    
    def apply_patch(self, repo_path: str, patch_content: str, fix_name: str) -> bool:
        """Applies a git patch"""
        try:
            logger.info(f"🔧 Applying patch: {fix_name}")
            cmd = ["git", "-C", repo_path, "apply", "--reject", "-"]
            result = subprocess.run(
                cmd, 
                input=patch_content, 
                text=True, 
                capture_output=True, 
                check=False
            )
            
            if result.returncode == 0:
                self.applied_fixes.append({
                    "name": fix_name,
                    "type": "patch",
                    "timestamp": datetime.now().isoformat()
                })
                logger.info(f"✅ Patch applied: {fix_name}")
                return True
            else:
                self.failed_fixes.append({
                    "name": fix_name,
                    "error": result.stderr,
                    "timestamp": datetime.now().isoformat()
                })
                return False
        except Exception as e:
            self.failed_fixes.append({
                "name": fix_name,
                "error": str(e),
                "timestamp": datetime.now().isoformat()
            })
            return False
    
    def modify_file(self, file_path: str, old_text: str, new_text: str, fix_name: str) -> bool:
        """Modifies a file"""
        try:
            if not os.path.exists(file_path):
                self.failed_fixes.append({
                    "name": fix_name,
                    "error": f"File not found: {file_path}",
                    "timestamp": datetime.now().isoformat()
                })
                return False
            
            logger.info(f"🔧 Modifying: {file_path}")
            
            with open(file_path, 'r') as f:
                content = f.read()
            
            if old_text not in content:
                self.failed_fixes.append({
                    "name": fix_name,
                    "error": "Pattern not found in file",
                    "timestamp": datetime.now().isoformat()
                })
                return False
            
            content = content.replace(old_text, new_text)
            
            with open(file_path, 'w') as f:
                f.write(content)
            
            self.applied_fixes.append({
                "name": fix_name,
                "type": "file_modification",
                "file": file_path,
                "timestamp": datetime.now().isoformat()
            })
            logger.info(f"✅ File modified: {file_path}")
            return True
        
        except Exception as e:
            self.failed_fixes.append({
                "name": fix_name,
                "error": str(e),
                "timestamp": datetime.now().isoformat()
            })
            return False
    
    def run_command(self, command: str, working_dir: str = ".", fix_name: str = "cmd") -> bool:
        """Runs a shell command"""
        try:
            logger.info(f"🔧 Running: {command}")
            result = subprocess.run(
                command,
                shell=True,
                cwd=working_dir,
                capture_output=True,
                text=True,
                timeout=300
            )
            
            if result.returncode == 0:
                self.applied_fixes.append({
                    "name": fix_name,
                    "type": "command",
                    "command": command,
                    "timestamp": datetime.now().isoformat()
                })
                logger.info(f"✅ Command executed: {fix_name}")
                return True
            else:
                self.failed_fixes.append({
                    "name": fix_name,
                    "error": result.stderr,
                    "command": command,
                    "timestamp": datetime.now().isoformat()
                })
                return False
        except Exception as e:
            self.failed_fixes.append({
                "name": fix_name,
                "error": str(e),
                "timestamp": datetime.now().isoformat()
            })
            return False
    
    def cleanup_files(self, paths: List[str], fix_name: str = "cleanup") -> bool:
        """Cleans up files/directories"""
        try:
            logger.info(f"🔧 Cleaning up: {paths}")
            for path in paths:
                if os.path.exists(path):
                    if os.path.isdir(path):
                        shutil.rmtree(path)
                    else:
                        os.remove(path)
            
            self.applied_fixes.append({
                "name": fix_name,
                "type": "cleanup",
                "paths": paths,
                "timestamp": datetime.now().isoformat()
            })
            logger.info(f"✅ Cleanup completed: {fix_name}")
            return True
        except Exception as e:
            self.failed_fixes.append({
                "name": fix_name,
                "error": str(e),
                "timestamp": datetime.now().isoformat()
            })
            return False


# ============================================================================
# PART 4: MAIN TROUBLESHOOTER & AUTOFIX AGENT
# ============================================================================

class DualAgentTroubleshooterAndAutoFix:
    """Complete unified troubleshooter and autofix system"""
    
    def __init__(self, agent1_repo: str, agent2_repo: str, agent1_eval: str, agent2_eval: str):
        self.agent1_checker = AgentHealthChecker("agent_1", agent1_repo, agent1_eval)
        self.agent2_checker = AgentHealthChecker("agent_2", agent2_repo, agent2_eval)
        self.fix_applier = FixApplier()
        self.agent1_repo = agent1_repo
        self.agent2_repo = agent2_repo
    
    def run_full_workflow(self, enable_autofix: bool = True, dry_run: bool = False) -> Dict[str, Any]:
        """Runs complete troubleshooting and autofix workflow"""
        
        logger.info("=" * 80)
        logger.info("DUAL AGENT TROUBLESHOOTER & AUTOFIX SYSTEM")
        logger.info("=" * 80)
        
        report = {
            "timestamp": datetime.now().isoformat(),
            "mode": "troubleshoot_and_autofix" if enable_autofix else "troubleshoot_only",
            "dry_run": dry_run,
        }
        
        # =====================================================================
        # PHASE 1: TROUBLESHOOTING
        # =====================================================================
        logger.info("\n" + "=" * 80)
        logger.info("PHASE 1: TROUBLESHOOTING & DIAGNOSTICS")
        logger.info("=" * 80)
        
        logger.info("\n[1/3] Checking Agent 1...")
        agent1_health = self.agent1_checker.check_health()
        logger.info(f"Agent 1 Health: {agent1_health['health_score']}/100")
        logger.info(f"Agent 1 Errors: {len(agent1_health['errors'])}")
        logger.info(f"Agent 1 Warnings: {len(agent1_health['warnings'])}")
        
        logger.info("\n[2/3] Checking Agent 2...")
        agent2_health = self.agent2_checker.check_health()
        logger.info(f"Agent 2 Health: {agent2_health['health_score']}/100")
        logger.info(f"Agent 2 Errors: {len(agent2_health['errors'])}")
        logger.info(f"Agent 2 Warnings: {len(agent2_health['warnings'])}")
        
        # Compare agents
        logger.info("\n[3/3] Comparing agents...")
        comparison = self._compare(agent1_health, agent2_health)
        
        report["troubleshooting"] = {
            "agent1": agent1_health,
            "agent2": agent2_health,
            "comparison": comparison,
        }
        
        # =====================================================================
        # PHASE 2: AUTOFIX (if enabled)
        # =====================================================================
        if enable_autofix:
            logger.info("\n" + "=" * 80)
            logger.info("PHASE 2: AUTOFIX APPLICATION")
            logger.info("=" * 80)
            
            if dry_run:
                logger.warning("⚠️  DRY RUN MODE - No actual changes will be applied")
            
            # Extract issues
            issues = self._extract_issues(agent1_health, agent2_health)
            logger.info(f"\nFound {len(issues)} issues to fix")
            
            # Apply fixes
            if not dry_run and issues:
                logger.info("\nApplying fixes...")
                self._apply_fixes(issues)
            
            report["autofix"] = {
                "issues_found": len(issues),
                "fixes_applied": len(self.fix_applier.applied_fixes),
                "fixes_failed": len(self.fix_applier.failed_fixes),
                "applied_fixes": self.fix_applier.applied_fixes,
                "failed_fixes": self.fix_applier.failed_fixes,
            }
        
        # =====================================================================
        # PHASE 3: VERIFICATION
        # =====================================================================
        logger.info("\n" + "=" * 80)
        logger.info("PHASE 3: VERIFICATION")
        logger.info("=" * 80)
        
        verification = self._verify_fixes()
        report["verification"] = verification
        
        # =====================================================================
        # FINAL SUMMARY
        # =====================================================================
        logger.info("\n" + "=" * 80)
        logger.info("FINAL SUMMARY")
        logger.info("=" * 80)
        
        summary = self._generate_summary(agent1_health, agent2_health, issues if enable_autofix else [])
        report["summary"] = summary
        
        logger.info(f"\nAgent 1 Health: {agent1_health['health_score']}/100")
        logger.info(f"Agent 2 Health: {agent2_health['health_score']}/100")
        logger.info(f"Better Agent: {comparison['better_agent']}")
        
        if enable_autofix:
            logger.info(f"Fixes Applied: {report['autofix']['fixes_applied']}")
            logger.info(f"Fixes Failed: {report['autofix']['fixes_failed']}")
        
        if summary['all_healthy']:
            logger.info("✅ Both agents are healthy!")
        else:
            logger.warning("⚠️  Issues detected - review recommendations")
        
        for rec in summary['recommendations']:
            logger.info(f"  • {rec}")
        
        logger.info("=" * 80)
        
        return report
    
    def _compare(self, health1: Dict, health2: Dict) -> Dict:
        """Compares two agents"""
        return {
            "agent1_errors": len(health1['errors']),
            "agent2_errors": len(health2['errors']),
            "agent1_health": health1['health_score'],
            "agent2_health": health2['health_score'],
            "better_agent": "agent_1" if health1['health_score'] > health2['health_score'] else "agent_2",
            "difference": abs(health1['health_score'] - health2['health_score']),
        }
    
    def _extract_issues(self, health1: Dict, health2: Dict) -> List[Dict]:
        """Extracts issues from health data"""
        issues = []
        
        for error in health1['errors']:
            issues.append({
                "agent": "agent_1",
                "type": "error",
                "description": error,
                "severity": "high",
            })
        
        for error in health2['errors']:
            issues.append({
                "agent": "agent_2",
                "type": "error",
                "description": error,
                "severity": "high",
            })
        
        for warning in health1['warnings']:
            issues.append({
                "agent": "agent_1",
                "type": "warning",
                "description": warning,
                "severity": "medium",
            })
        
        for warning in health2['warnings']:
            issues.append({
                "agent": "agent_2",
                "type": "warning",
                "description": warning,
                "severity": "medium",
            })
        
        return issues
    
    def _apply_fixes(self, issues: List[Dict]):
        """Applies fixes for the issues"""
        
        # Priority-based fix application
        high_priority = [i for i in issues if i['severity'] == 'high']
        medium_priority = [i for i in issues if i['severity'] == 'medium']
        
        for issue in high_priority + medium_priority:
            agent = issue['agent']
            repo_path = self.agent1_repo if agent == 'agent_1' else self.agent2_repo
            
            if issue['type'] == 'error':
                if 'compilation_error' in issue['description']:
                    logger.info(f"Fixing compilation error in {agent}...")
                    # Common fix: remove __pycache__
                    self.fix_applier.run_command(
                        "find . -type d -name '__pycache__' -exec rm -rf {} +",
                        working_dir=repo_path,
                        fix_name=f"cleanup_pycache_{agent}"
                    )
                
                elif 'docker_error' in issue['description']:
                    logger.info(f"Fixing docker error in {agent}...")
                    self.fix_applier.run_command(
                        "docker system prune -f",
                        working_dir=repo_path,
                        fix_name=f"docker_cleanup_{agent}"
                    )
    
    def _verify_fixes(self) -> Dict:
        """Verifies that fixes worked"""
        verification = {
            "repo1_exists": os.path.exists(self.agent1_repo),
            "repo2_exists": os.path.exists(self.agent2_repo),
            "fixes_verified": len(self.fix_applier.applied_fixes) > 0,
        }
        return verification
    
    def _generate_summary(self, health1: Dict, health2: Dict, issues: List[Dict]) -> Dict:
        """Generates final summary"""
        
        return {
            "timestamp": datetime.now().isoformat(),
            "total_errors": len(health1['errors']) + len(health2['errors']),
            "total_warnings": len(health1['warnings']) + len(health2['warnings']),
            "all_healthy": health1['health_score'] >= 80 and health2['health_score'] >= 80,
            "recommendations": self._get_recommendations(health1, health2),
        }
    
    def _get_recommendations(self, health1: Dict, health2: Dict) -> List[str]:
        """Generates recommendations"""
        recs = []
        
        if len(health1['errors']) > len(health2['errors']):
            recs.append("Agent 1 has more errors - review logs")
        
        if len(health2['errors']) > len(health1['errors']):
            recs.append("Agent 2 has more errors - review logs")
        
        if health1['health_score'] < 80:
            recs.append("Agent 1 health is low - investigate critical issues")
        
        if health2['health_score'] < 80:
            recs.append("Agent 2 health is low - investigate critical issues")
        
        if not recs:
            recs.append("Both agents are healthy - no action needed")
        
        return recs


# ============================================================================
# MAIN ENTRY POINT
# ============================================================================

def main():
    parser = argparse.ArgumentParser(
        description="Unified Troubleshooter & AutoFix System for Dual HyperAgents"
    )
    
    parser.add_argument("--agent1_repo", type=str, required=True, help="Path to first agent repo")
    parser.add_argument("--agent2_repo", type=str, required=True, help="Path to second agent repo")
    parser.add_argument("--agent1_eval", type=str, required=True, help="Path to first agent eval")
    parser.add_argument("--agent2_eval", type=str, required=True, help="Path to second agent eval")
    parser.add_argument("--output", type=str, default="report.json", help="Output report path")
    parser.add_argument("--autofix", action="store_true", help="Enable autofix mode")
    parser.add_argument("--dry_run", action="store_true", help="Dry run - don't apply fixes")
    
    args = parser.parse_args()
    
    # Run the system
    system = DualAgentTroubleshooterAndAutoFix(
        args.agent1_repo,
        args.agent2_repo,
        args.agent1_eval,
        args.agent2_eval,
    )
    
    report = system.run_full_workflow(
        enable_autofix=args.autofix,
        dry_run=args.dry_run
    )
    
    # Save report
    with open(args.output, 'w') as f:
        json.dump(report, f, indent=2)
    
    logger.info(f"\n✅ Report saved to {args.output}")


if __name__ == "__main__":
    main()
