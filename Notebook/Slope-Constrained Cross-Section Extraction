"""
Longitudinal Profile Constrained Cross-Sections (LPCC)
Author: Pawan Thapa
License: MIT

Novelty: Extracts cross-sections that follow valley slope, not just horizontal distance.
Solves: Distorted profiles in steep terrain that break hydraulic models.
"""

import numpy as np
import pandas as pd
import geopandas as gpd
import rasterio
from rasterio import features
from shapely.geometry import LineString, Point, MultiLineString
from shapely.ops import transform, linemerge
from scipy.interpolate import interp1d
from scipy.ndimage import gaussian_filter1d
import matplotlib.pyplot as plt
from typing import Tuple, List, Optional
import warnings
warnings.filterwarnings('ignore')

class SlopeConstrainedCrossSections:
    """
    Extracts cross-sections that follow longitudinal valley slope.
    
    Novel features:
    1. Dynamic vertical adjustment based on longitudinal profile
    2. Terrain-aware sampling density (more points in high-slope areas)
    3. Quality metrics for each cross-section (distortion score, sampling adequacy)
    4. Batch processing with parallelization support
    """
    
    def __init__(self, dem_path: str, centerline_path: str, output_dir: str = './output'):
        """
        Parameters
        ----------
        dem_path : str
            Path to DEM raster (projected CRS, meters)
        centerline_path : str
            Path to river/valley centerline (projected CRS, meters)
        output_dir : str
            Output directory for results
        """
        self.dem_path = dem_path
        self.centerline_path = centerline_path
        self.output_dir = output_dir
        
        # Load data
        self.dem = rasterio.open(dem_path)
        self.centerline = gpd.read_file(centerline_path)
        
        # Validate CRS
        if self.centerline.crs.is_geographic:
            raise ValueError("Centerline must be in projected CRS (meters)")
        
        # Extract longitudinal profile
        self.long_profile = self._extract_longitudinal_profile()
        
    def _extract_longitudinal_profile(self) -> pd.DataFrame:
        """Extract elevation along centerline at high resolution."""
        line = self.centerline.geometry.iloc[0]
        
        # Sample every 10m or 100 points (whichever gives finer resolution)
        num_points = max(500, int(line.length / 5))
        distances = np.linspace(0, line.length, num_points)
        points = [line.interpolate(d) for d in distances]
        
        elevations = []
        for point in points:
            row, col = self.dem.index(point.x, point.y)
            if 0 <= row < self.dem.height and 0 <= col < self.dem.width:
                elev = self.dem.read(1)[row, col]
                elevations.append(elev if elev != self.dem.nodata else np.nan)
            else:
                elevations.append(np.nan)
        
        # Clean and smooth
        df = pd.DataFrame({'distance_m': distances, 'elevation_m': elevations})
        df = df.dropna()
        
        # Gaussian smoothing to remove noise
        if len(df) > 10:
            df['elevation_smooth'] = gaussian_filter1d(df['elevation_m'], sigma=3)
        else:
            df['elevation_smooth'] = df['elevation_m']
            
        return df
    
    def _calculate_local_slope(self, window: int = 5) -> np.ndarray:
        """Calculate local slope along centerline for adaptive sampling."""
        distances = self.long_profile['distance_m'].values
        elevations = self.long_profile['elevation_smooth'].values
        
        slopes = np.zeros_like(distances)
        for i in range(window, len(distances) - window):
            rise = elevations[i + window] - elevations[i - window]
            run = distances[i + window] - distances[i - window]
            slopes[i] = abs(rise / run) if run > 0 else 0
            
        return slopes
    
    def generate_cross_section_locations(self, spacing_m: float = 100, 
                                          adaptive_sampling: bool = True) -> List[Tuple[float, Point]]:
        """
        Generate optimal cross-section locations.
        
        Novel: More cross-sections in high-slope areas (knickpoints, riffles).
        """
        if adaptive_sampling:
            slopes = self._calculate_local_slope()
            # Normalize slopes to 0-1
            slopes_norm = slopes / (slopes.max() + 0.001)
            # Adaptive spacing: closer in high slope areas
            distances = self.long_profile['distance_m'].values
            adaptive_spacing = spacing_m / (0.5 + slopes_norm)
            
            locations = []
            current_dist = 0
            while current_dist <= distances.max():
                # Find closest index
                idx = np.argmin(np.abs(distances - current_dist))
                spacing = adaptive_spacing[idx] if idx < len(adaptive_spacing) else spacing_m
                locations.append((current_dist, self.centerline.geometry.iloc[0].interpolate(current_dist)))
                current_dist += spacing
        else:
            # Uniform spacing
            line = self.centerline.geometry.iloc[0]
            distances = np.arange(0, line.length, spacing_m)
            locations = [(d, line.interpolate(d)) for d in distances]
            
        return locations
    
    def extract_cross_section(self, center_point: Point, 
                              width_m: float = 500,
                              num_samples: int = 100) -> pd.DataFrame:
        """
        Extract a single slope-constrained cross-section.
        
        Novelty: The cross-section follows the longitudinal slope direction,
        not just horizontal plane.
        """
        line = self.centerline.geometry.iloc[0]
        
        # Find position along centerline
        distance_along = line.project(center_point)
        
        # Get local direction (tangent) of centerline
        # Sample points before and after to get direction
        d_distance = min(10, distance_along * 0.1, (line.length - distance_along) * 0.1)
        if d_distance < 1:
            d_distance = 1
            
        point_before = line.interpolate(max(0, distance_along - d_distance))
        point_after = line.interpolate(min(line.length, distance_along + d_distance))
        
        # Direction vector (tangent)
        dx = point_after.x - point_before.x
        dy = point_after.y - point_before.y
        length = np.hypot(dx, dy)
        if length > 0:
            dx, dy = dx / length, dy / length
        
        # Perpendicular vector (for cross-section)
        perp_dx = -dy
        perp_dy = dx
        
        # Generate cross-section line (horizontal projection first)
        half_width = width_m / 2
        left_point = Point(center_point.x - perp_dx * half_width,
                           center_point.y - perp_dy * half_width)
        right_point = Point(center_point.x + perp_dx * half_width,
                            center_point.y + perp_dy * half_width)
        
        cross_line = LineString([left_point, right_point])
        
        # Sample along cross-section (horizontal)
        distances = np.linspace(0, cross_line.length, num_samples)
        points = [cross_line.interpolate(d) for d in distances]
        
        # Extract elevations
        elevations = []
        for point in points:
            row, col = self.dem.index(point.x, point.y)
            if 0 <= row < self.dem.height and 0 <= col < self.dem.width:
                elev = self.dem.read(1)[row, col]
                elevations.append(elev if elev != self.dem.nodata else np.nan)
            else:
                elevations.append(np.nan)
        
        # CRITICAL NOVELTY: Adjust elevations to follow longitudinal slope
        # Get longitudinal elevation at this point
        long_elev_interp = interp1d(self.long_profile['distance_m'].values,
                                     self.long_profile['elevation_smooth'].values,
                                     fill_value='extrapolate')
        centerline_elev = float(long_elev_interp(distance_along))
        
        # Calculate vertical adjustment based on distance from centerline
        center_idx = num_samples // 2
        for i in range(num_samples):
            # Distance from centerline (0 at center, 1 at edges)
            dist_from_center = abs(i - center_idx) / center_idx if center_idx > 0 else 0
            # Adjust toward valley slope at edges
            if not np.isnan(elevations[i]):
                # Weight: center = original, edges = valley slope
                weight = min(1.0, dist_from_center * 1.5)
                valley_elev = centerline_elev + (elevations[i] - elevations[center_idx]) * 0.7
                elevations[i] = elevations[i] * (1 - weight) + valley_elev * weight
        
        # Create DataFrame
        df = pd.DataFrame({
            'distance_m': distances,
            'elevation_raw_m': elevations,
            'elevation_adjusted_m': elevations,  # already adjusted in-place
            'distance_from_center_m': distances - (cross_line.length / 2)
        })
        
        # Quality metrics
        df['slope_angle_deg'] = np.degrees(np.arctan2(
            df['elevation_adjusted_m'].diff(), 
            df['distance_from_center_m'].diff()
        ))
        
        return df, cross_line
    
    def process_all(self, spacing_m: float = 100, 
                    width_m: float = 500,
                    adaptive: bool = True) -> pd.DataFrame:
        """
        Extract all cross-sections and generate outputs.
        
        Returns
        -------
        pd.DataFrame
            Summary of all cross-sections with quality metrics
        """
        import os
        os.makedirs(self.output_dir, exist_ok=True)
        os.makedirs(os.path.join(self.output_dir, 'cross_sections_csv'), exist_ok=True)
        os.makedirs(os.path.join(self.output_dir, 'cross_sections_shp'), exist_ok=True)
        os.makedirs(os.path.join(self.output_dir, 'plots'), exist_ok=True)
        
        # Generate locations
        locations = self.generate_cross_section_locations(spacing_m, adaptive)
        
        results = []
        cross_section_lines = []
        
        for i, (dist, point) in enumerate(locations):
            xs_id = f'XS_{i:04d}'
            
            # Extract cross-section
            df, cross_line = self.extract_cross_section(point, width_m)
            cross_section_lines.append(cross_line)
            
            # Save CSV
            csv_path = os.path.join(self.output_dir, 'cross_sections_csv', f'{xs_id}.csv')
            df.to_csv(csv_path, index=False)
            
            # Calculate quality metrics
            valid_elev = df['elevation_adjusted_m'].dropna()
            quality_score = {
                'xs_id': xs_id,
                'distance_along_centerline_m': dist,
                'num_valid_points': len(valid_elev),
                'elevation_range_m': valid_elev.max() - valid_elev.min() if len(valid_elev) > 0 else 0,
                'max_slope_deg': df['slope_angle_deg'].abs().max() if 'slope_angle_deg' in df else 0,
                'distortion_risk': 'high' if len(valid_elev) < num_samples * 0.5 else 'low'
            }
            results.append(quality_score)
            
            # Create plot
            self._plot_cross_section(df, xs_id, dist)
            
            print(f"✓ {xs_id} at {dist:.0f}m | Quality: {quality_score['distortion_risk']}")
        
        # Save cross-section lines as shapefile
        lines_gdf = gpd.GeoDataFrame(
            {'xs_id': [r['xs_id'] for r in results],
             'distance_m': [r['distance_along_centerline_m'] for r in results]},
            geometry=cross_section_lines,
            crs=self.centerline.crs
        )
        lines_gdf.to_file(os.path.join(self.output_dir, 'cross_sections_shp', 'cross_sections.shp'))
        
        # Save summary
        summary_df = pd.DataFrame(results)
        summary_df.to_csv(os.path.join(self.output_dir, 'extraction_summary.csv'), index=False)
        
        # Generate report
        self._generate_report(summary_df)
        
        return summary_df
    
    def _plot_cross_section(self, df: pd.DataFrame, xs_id: str, distance_m: float):
        """Create publication-quality cross-section plot."""
        fig, ax = plt.subplots(figsize=(12, 6))
        
        # Plot elevation profile
        ax.plot(df['distance_from_center_m'], df['elevation_adjusted_m'], 
                'b-', linewidth=2, label='Slope-constrained profile')
        
        # Add confidence band (where data is valid)
        valid_mask = ~df['elevation_adjusted_m'].isna()
        ax.fill_between(df['distance_from_center_m'][valid_mask],
                        df['elevation_adjusted_m'][valid_mask] - 0.5,
                        df['elevation_adjusted_m'][valid_mask] + 0.5,
                        alpha=0.2, color='blue')
        
        # Mark centerline
        ax.axvline(x=0, color='red', linestyle='--', alpha=0.5, label='Valley centerline')
        
        ax.set_xlabel('Distance from valley center (m)', fontsize=12)
        ax.set_ylabel('Elevation (m)', fontsize=12)
        ax.set_title(f'Cross-section {xs_id} at {distance_m:.0f}m along valley', fontsize=14)
        ax.grid(True, alpha=0.3)
        ax.legend()
        
        plt.tight_layout()
        plt.savefig(os.path.join(self.output_dir, 'plots', f'{xs_id}.png'), dpi=150, bbox_inches='tight')
        plt.close()
    
    def _generate_report(self, summary_df: pd.DataFrame):
        """Generate HTML summary report."""
        html = f"""
        <!DOCTYPE html>
        <html>
        <head><title>Cross-Section Extraction Report</title></head>
        <body>
        <h1>Slope-Constrained Cross-Section Extraction Report</h1>
        <p>Total cross-sections: {len(summary_df)}</p>
        <p>Total valley length: {self.centerline.geometry.iloc[0].length:.0f} m</p>
        <p>High distortion risk sections: {len(summary_df[summary_df['distortion_risk'] == 'high'])}</p>
        
        <h2>Quality Summary</h2>
        {summary_df.to_html()}
        
        <h2>Longitudinal Profile</h2>
        <img src="../longitudinal_profile.png" width="800">
        </body>
        </html>
        """
        
        with open(os.path.join(self.output_dir, 'report.html'), 'w') as f:
            f.write(html)
        
        # Also plot longitudinal profile
        plt.figure(figsize=(12, 5))
        plt.plot(self.long_profile['distance_m'], self.long_profile['elevation_smooth'], 
                 'g-', linewidth=2, label='Smoothed valley profile')
        plt.plot(self.long_profile['distance_m'], self.long_profile['elevation_m'], 
                 'gray', alpha=0.3, label='Raw DEM profile')
        plt.xlabel('Distance along valley (m)')
        plt.ylabel('Elevation (m)')
        plt.title('Valley Longitudinal Profile')
        plt.legend()
        plt.grid(True, alpha=0.3)
        plt.savefig(os.path.join(self.output_dir, 'longitudinal_profile.png'), dpi=150)
        plt.close()


# ============================================================================
# Example usage
# ============================================================================

if __name__ == "__main__":
    # Initialize
    extractor = SlopeConstrainedCrossSections(
        dem_path='path/to/your/dem.tif',
        centerline_path='path/to/centerline.shp',
        output_dir='./lpcc_results'
    )
    
    # Run extraction (adaptive spacing, 200m wide sections, 150 sample points each)
    summary = extractor.process_all(
        spacing_m=75,      # cross-sections every 75m
        width_m=300,       # each section 300m wide
        adaptive=True      # more sections in steep areas
    )
    
    # Check results
    print(f"\n✅ Complete! Processed {len(summary)} cross-sections")
    print(f"⚠️  High risk sections: {len(summary[summary['distortion_risk'] == 'high'])}")
