# troubleshooter.py
# Simple troubleshooting agent for dual HyperAgents

import argparse
import json
import logging
import os
import re
import subprocess
from datetime import datetime
from typing import Dict, List, Tuple, Any

# Setup logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)


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


class DualAgentTroubleshooter:
    """Main troubleshooter for dual agents"""
    
    def __init__(self, agent1_repo: str, agent2_repo: str, agent1_eval: str, agent2_eval: str):
        self.agent1_checker = AgentHealthChecker("agent_1", agent1_repo, agent1_eval)
        self.agent2_checker = AgentHealthChecker("agent_2", agent2_repo, agent2_eval)
    
    def troubleshoot(self) -> Dict[str, Any]:
        """Runs troubleshooting on both agents"""
        
        logger.info("=" * 80)
        logger.info("DUAL AGENT TROUBLESHOOTER")
        logger.info("=" * 80)
        
        # Check both agents
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
        
        # Build report
        report = {
            "timestamp": datetime.now().isoformat(),
            "agent1": agent1_health,
            "agent2": agent2_health,
            "comparison": comparison,
            "summary": self._generate_summary(agent1_health, agent2_health),
        }
        
        # Print summary
        logger.info("\n" + "=" * 80)
        logger.info("SUMMARY")
        logger.info("=" * 80)
        logger.info(f"Agent 1 Health: {agent1_health['health_score']}/100")
        logger.info(f"Agent 2 Health: {agent2_health['health_score']}/100")
        logger.info(f"Better Agent: {comparison['better_agent']}")
        if agent1_health['errors'] or agent2_health['errors']:
            logger.warning("⚠️  Issues detected - see report for details")
        else:
            logger.info("✅ Both agents are healthy!")
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
    
    def _generate_summary(self, health1: Dict, health2: Dict) -> Dict:
        """Generates summary"""
        return {
            "timestamp": datetime.now().isoformat(),
            "total_errors": len(health1['errors']) + len(health2['errors']),
            "total_warnings": len(health1['warnings']) + len(health2['warnings']),
            "agents_healthy": health1['health_score'] >= 80 and health2['health_score'] >= 80,
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
            recs.append("Agent 1 health is low - investigate issues")
        
        if health2['health_score'] < 80:
            recs.append("Agent 2 health is low - investigate issues")
        
        if not recs:
            recs.append("Both agents are performing well - no action needed")
        
        return recs


def main():
    parser = argparse.ArgumentParser(
        description="Troubleshooter for dual HyperAgent instances"
    )
    
    parser.add_argument(
        "--agent1_repo",
        type=str,
        required=True,
        help="Path to first agent repository"
    )
    parser.add_argument(
        "--agent2_repo",
        type=str,
        required=True,
        help="Path to second agent repository"
    )
    parser.add_argument(
        "--agent1_eval",
        type=str,
        required=True,
        help="Path to first agent evaluation outputs"
    )
    parser.add_argument(
        "--agent2_eval",
        type=str,
        required=True,
        help="Path to second agent evaluation outputs"
    )
    parser.add_argument(
        "--output",
        type=str,
        default="troubleshoot_report.json",
        help="Output report path"
    )
    
    args = parser.parse_args()
    
    # Run troubleshooter
    troubleshooter = DualAgentTroubleshooter(
        args.agent1_repo,
        args.agent2_repo,
        args.agent1_eval,
        args.agent2_eval,
    )
    
    report = troubleshooter.troubleshoot()
    
    # Save report
    with open(args.output, 'w') as f:
        json.dump(report, f, indent=2)
    
    logger.info(f"\n✅ Report saved to {args.output}")


if __name__ == "__main__":
    main()
